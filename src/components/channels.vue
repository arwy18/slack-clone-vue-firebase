<template>
    <div>
        <button type="button" class="btn btn-outline-primary btn-sm" v-on:click="openModal()">Add a new channel</button>
        <div class="mt-4 mb-4">
            <button v-for="(channel, index) in channels" :key="index"
                    v-bind:class="[isActiveChannel(channel)?
                'list-group-item list-group-item-action active' :
                'list-group-item list-group-item-action']"
                    v-on:click.stop="setActiveChannel(channel)">
                {{ channel.name }}
                <span class="float-right"> {{ getNotificationsCount(channel) || '' }}</span>
            </button>
        </div>
        <!-- Modal -->
        <div class="modal fade" id="channelModal" data-backdrop="static" tabindex="-1" role="dialog">
            <div class="modal-dialog modal-dialog-centered">
                <div class="modal-content rounded-0">
                    <div class="modal-header">
                        <h5 class="modal-title">Add a new channel</h5>
                        <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                            <span aria-hidden="true">&times;</span>
                        </button>
                    </div>
                    <div class="modal-body">
                        <div class="form-group">
                            <input type="text" v-model.trim="newChannelName" v-uppercase class="form-control" id="newChannel" name="newChannel">
                        </div>
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-dismiss="modal" v-on:click.stop="closeModal()">Cancel</button>
                        <button type="button" class="btn btn-primary" v-on:click.stop="saveChannelToRef()">Accept</button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import utils from '../assets/js/utils'
    import $ from 'jquery'

    export default {
        name: 'channels',
        data() {
            return {
                channelsRef: window.firebase.database().ref('channels'),
                messagesRef: window.firebase.database().ref('messages'),
                channels: [],
                notifications: [],
                newChannelName: null
            }
        },
        mounted: function () {
            const self = this
            self.addListeners()
        },
        beforeDestroy: function () {
            const self = this
            self.removeAllListeners()
        },
        computed: {
            currentUser: function () {
                const self = this
                return self.$store.getters.getCurrentUser
            },
            currentChannel: function () {
                const self = this
                return self.$store.getters.getCurrentChannel
            }
        },
        methods: {
            openModal: function () {
                $('#channelModal').modal('show')
            },
            closeModal: function () {
                const self = this
                $('#channelModal').modal('hide')
                self.newChannelName = null
            },
            saveChannelToRef: function () {
                const self = this
                // #region VALIDATIONS
                if (utils.isNullOrEmpty(self.newChannelName)) {
                    return
                }
                // #endregion

                var key = self.channelsRef.push().key
                var newChannel = {
                    id: utils.tryGet(() => key),
                    name: utils.tryGet(() => self.newChannelName.toUpperCase()) || null,
                    private: false
                }

                self.channelsRef.child(key).update(newChannel)
                    .then(function () {
                        self.newChannelName = null
                        self.$store.dispatch('setCurrentChannel', newChannel)
                        $('#channelModal').modal('hide')
                    })
                    .catch(function (error) {
                        self.$toast.error(error.message, { position: 'top' })
                    })
            },
            addListeners: function () {
                const self = this
                self.channelsRef.orderByChild("name").on('child_added', function (snapshot) {
                    var channel = snapshot.val()
                    if (!utils.isNullOrEmpty(channel.name)) {

                        self.addNotificationListener(channel)
                        self.channels.push(channel)
                        self.channels.sort(function (a, b) {
                            return a.name.localeCompare(b.name)
                        })

                        if (self.channels.length && !self.currentChannel) {
                            self.setActiveChannel(self.channels[0])
                        }
                    }
                })
            },
            addNotificationListener: function (channel) {
                const self = this
                var channelId = channel.id
                self.messagesRef.child(channelId).on('value', function (snapshot) {
                    var index = self.notifications.findIndex(x => x.id === channelId)
                    if (index === -1) {
                        self.notifications.push({ id: channelId, total: 0, totalChildren: 0 })
                    }
                    else {
                        var totalChildren = snapshot.numChildren()
                        if (channelId !== self.currentChannel.id &&
                            (totalChildren > self.notifications[index].totalChildren)) {
                            self.notifications[index].total += 1
                            self.notifications[index].totalChildren = totalChildren
                        }
                    }
                })
            },
            removeAllListeners: function () {
                const self = this
                self.channelsRef.off()
                self.channels.forEach(x => self.messagesRef.child(x.id).off())
            },
            getNotificationsCount: function (channel) {
                const self = this
                return utils.tryGet(() => {
                    var notification = self.notifications.find(x => x.id === channel.id)
                    return notification.total
                })
            },
            isActiveChannel: function (channel) {
                const self = this
                return utils.tryGet(() => self.currentChannel.id === channel.id)
            },
            setActiveChannel: function (channel) {
                const self = this
                self.$store.dispatch('setCurrentChannel', channel)
                var index = self.notifications.findIndex(x => x.id === channel.id)
                if (index !== -1) {
                    self.notifications[index].total = 0
                    self.notifications[index].totalChildren = 0
                }
            }
        }
    }
</script>
