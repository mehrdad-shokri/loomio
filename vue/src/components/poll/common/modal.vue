<script lang="coffee">
import Records from '@/shared/services/records'
import EventBus from '@/shared/services/event_bus'
import Flash  from '@/shared/services/flash'
import { onError } from '@/shared/helpers/form'

export default
  props:
    poll: Object
    close: Function

  computed:
    title_key: ->
      mode = if @poll.isNew()
        'start'
      else
        'edit'
      'poll_' + @poll.pollType + '_form.'+mode+'_header'

    isEditing: ->
      !@poll.isNew()

  methods:
    submit: ->
      actionName = if @poll.isNew() then 'created' else 'updated'
      @poll.customFields.deanonymize_after_close = @poll.deanonymizeAfterClose if @poll.anonymous
      @poll.customFields.can_respond_maybe = @poll.canRespondMaybe if @poll.pollType == 'meeting'
      @poll.setErrors({})
      @poll.save()
      .then (data) =>
        pollKey = data.polls[0].key
        Records.polls.findOrFetchById(pollKey, {}, true).then (poll) =>
          if !@poll.discussionId
            @$router.replace(@urlFor(poll)).catch (err) => {}
          Flash.success "poll_#{poll.pollType}_form.#{poll.pollType}_#{actionName}"
          @close()
          return if actionName == 'updated'
          EventBus.$emit 'openModal',
            component: 'AnnouncementForm',
            props: { announcement: Records.announcements.buildFromModel(poll) }

      .catch onError(@poll)

</script>
<template lang="pug">
v-card.poll-common-modal(@keyup.ctrl.enter="submit()" @keydown.meta.enter.stop.capture="submit()")
  submit-overlay(:value="poll.processing")
  v-card-title
    h1.headline(tabindex="-1" v-t="title_key")
    v-spacer
    dismiss-modal-button(:close='close')
  v-card-text
    v-alert(v-model="isEditing" color="primary" type="warning")
      template(slot="default")
        span(v-t="{ path: 'poll_common.edit_warning', args: { pollType: poll.translatedPollType()}}")
    poll-common-directive(:poll='poll', name='form', :modal='true')
  v-card-actions.poll-common-form-actions
    v-spacer
    v-btn.poll-common-form__submit(color="primary" @click='submit()', v-if='!poll.isNew()', v-t="'common.action.save_changes'")
    v-btn.poll-common-form__submit(color="primary" @click='submit()', v-if='poll.isNew() && poll.groupId' v-t="{path: 'poll_common_form.start_poll_type', args: {poll_type: poll.translatedPollType()}}")
</template>
