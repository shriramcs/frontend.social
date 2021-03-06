<template>
  <b-container class="meeting-container">
    <b-breadcrumb :items="items" />
    <b-row>
      <b-col md="12">
        <b-row>
          <div class="zoom-container">
            <iframe
              v-if="zoomUrl"
              :src="zoomUrl"
            />
          </div>
        </b-row>
        <b-row style="margin-top: 20px">
          <h1>Group Topics (Click to Join call)</h1>
        </b-row>
        <b-row>
          <b-col md="12">
            <EventMeetings
              :id="eventId || userId"
              :type="eventId ? 'EVENT' : 'USER'"
              :is-editable="isAdmin"
              :admins="event ? event.adminUsers : []"
            />
          </b-col>
        </b-row>

        <b-row
          v-if="!hideComments"
          style="margin-top: 20px"
        >
          <h1>Comments</h1>
        </b-row>
        <b-row v-if="!hideComments && signedInUser">
          <b-col md="12">
            <add-comment
              ref="addcomment"
              :comment-id="commentId"
              :on-save="saveComment"
              :on-cancel="cancelComment"
              :show-rating="showRating"
              :parent-id="meetingId"
              class="mt-1"
            />
          </b-col>
        </b-row>
        <b-row
          v-if="!hideComments"
          style="margin-bottom: 20px"
        >
          <b-col md="12">
            <Comment
              v-for="(comment, index) in comments"
              :key="comment._id"
              :index="index"
              :comment-id="comment._id"
              :comment="comment"
              :show-rating="showRating"
              :allow-reply="allowReply"
              :on-delete="deleteComment"
              :on-edit="editComment"
              :on-save="saveComment"
            />
          </b-col>
        </b-row>
      </b-col>
      <!-- <b-col md="3">
        <OnlineUsers />
      </b-col> -->
    </b-row>
  </b-container>
</template>

<script>
import eventService from '@/services/event.service';
import EventMeetings from '@/components/Events/EventMeetings.vue';
// import OnlineUsers from "@/components/OnlineUsers/OnlineUsers.vue";

import commentService from '@/services/comment.service';
import UserService from '@/services/user.service';
import Comment from '@/components/Comment/Comment.vue';
import AddComment from '@/components/Comment/AddComment.vue';
import eventBus from '@/utilities/eventBus';
import { ToastType, messages, INTERVALS } from '@/constants/constants';
import { isEmpty } from 'lodash';

export default {
	components: {
		EventMeetings,
		// OnlineUsers,
		Comment,
		AddComment,
	},
	data() {
		return {
			eventTitle: '',
			roomOwner: {},
			eventId: '',
			groupTopic: '',
			userId: '',
			comments: [],
			showRating: false,
			allowReply: false,
			commentId: '',
			hideComments: false,
			interval: null,
			event: {},
		};
	},
	computed: {
		zoomUrl() {
			const username = this.signedInUser
				? this.signedInUser.username
				: this.guestUser
				? this.guestUser.name
				: null;
			return username
				? `/jitsi.html?id=${this.$route.params.id}&name=${username}&t=${
						this.isAdmin ? 1 : 0
				  }&private=${this.userId ? 1 : 0}`
				: '';
		},
		signedInUser() {
			return this.$store.state.signedInUser;
		},
		guestUser() {
			return this.$store.state.guestUser;
		},
		isAdmin() {
			if (!this.signedInUser) return false;

			const username = this.signedInUser.username.toLowerCase();
			if (!isEmpty(this.event)) {
				const adminUsers = this.event.adminUsers || [];
				return (
					this.event.createdBy.username.toLowerCase() === username ||
					adminUsers.some((x) => x.username.toLowerCase() === username)
				);
			} else if (!isEmpty(this.roomOwner)) {
				return username === this.roomOwner.username.toLowerCase();
			}
			return false;
		},
		items() {
			if (this.eventId) {
				return [
					{
						text: 'Events',
						to: '/events',
					},
					{
						text: this.eventTitle,
						to: `/event/${this.event.uniqueId}`,
					},
					{
						text: this.groupTopic,
						active: true,
					},
				];
			} else {
				return [
					{
						text: `${this.roomOwner.name}'s Video Rooms`,
						to:
							this.signedInUser &&
							this.signedInUser.username === this.roomOwner.username
								? '/me'
								: `/user/${this.roomOwner.username}`,
					},
					{
						text: this.groupTopic,
						active: true,
					},
				];
			}
		},
		meetingId() {
			//This will be used into the database to save and fetch comments
			return this.$route.params.id;
		},
	},
	watch: {
		$route() {
			this.updateBreadcrumb();
		},
	},
	created() {
		this.updateBreadcrumb();
	},
	mounted() {
		setTimeout(() => {
			if (this.signedInUser == null && this.guestUser == null) {
				this.$router.push('/signin');
				return;
			}
		}, 1000);
	},
	beforeDestroy() {
		if (this.interval) {
			clearInterval(this.interval);
		}
	},
	methods: {
		updateBreadcrumb() {
			let urlParams = new URLSearchParams(window.location.search);
			this.groupTopic = urlParams.get('title');
			this.eventId = urlParams.get('eventId');
			this.userId = urlParams.get('userId');
			if (this.eventId) {
				eventService.getEventById(this.eventId).then((event) => {
					this.eventTitle = event.title;
					this.event = event;
				});
			} else if (this.userId) {
				UserService.getUserByUserId(this.userId).then((users) => {
					if (users.length) {
						this.roomOwner = users[0];
					}
				});
			}
			this.getComments();
			this.interval = setInterval(
				() => this.getComments(),
				INTERVALS.TwentySeconds
			);
		},
		getComments() {
			commentService
				.getComment(this.meetingId)
				.then((response) => {
					this.comments = response;
				})
				.catch(() => {
					eventBus.$emit('show-toast', {
						body: e.message,
						title: messages.generic.error,
						type: ToastType.ERROR,
					});
				});
		},
		saveComment(response, index) {
			if (this.commentId != '') {
				this.comments.splice(index, 1, response);
				this.commentId = '';
			} else {
				this.comments.push(response);
			}

			eventBus.$emit('show-toast', {
				body: messages.comment.commentAddSuccess,
				title: messages.generic.success,
			});
		},
		deleteComment(index) {
			console.log(index);
			this.comments.splice(index, 1);
		},
		editComment(commentId) {
			this.commentId = commentId;
		},
		cancelComment() {},
	},
};
</script>
<style lang="scss" scoped>
.meeting-container {
	padding-left: 5%;
}
.zoom-container {
	width: 100%;
	height: 100%;
	iframe {
		height: 600px;
		width: 100%;
	}
}
</style>
