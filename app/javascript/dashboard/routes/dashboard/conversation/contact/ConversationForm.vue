<template>
  <form class="conversation--form w-full" @submit.prevent="onFormSubmit">
    <div v-if="showNoInboxAlert" class="callout warning">
      <p>
        {{ $t('NEW_CONVERSATION.NO_INBOX') }}
      </p>
    </div>
    <div v-else>
      <div class="gap-2 flex flex-row">
        <div class="w-[50%]">
          <label>
            {{ $t('NEW_CONVERSATION.FORM.INBOX.LABEL') }}
          </label>
          <div class="multiselect-wrap--small">
            <multiselect
              v-model="targetInbox"
              track-by="id"
              label="name"
              :placeholder="$t('FORMS.MULTISELECT.SELECT')"
              selected-label=""
              select-label=""
              deselect-label=""
              :max-height="160"
              :close-on-select="true"
              :options="[...inboxes]"
            >
              <template slot="singleLabel" slot-scope="{ option }">
                <inbox-dropdown-item
                  v-if="option.name"
                  :name="option.name"
                  :inbox-identifier="computedInboxSource(option)"
                  :channel-type="option.channel_type"
                />
                <span v-else>
                  {{ $t('NEW_CONVERSATION.FORM.INBOX.PLACEHOLDER') }}
                </span>
              </template>
              <template slot="option" slot-scope="{ option }">
                <inbox-dropdown-item
                  :name="option.name"
                  :inbox-identifier="computedInboxSource(option)"
                  :channel-type="option.channel_type"
                />
              </template>
            </multiselect>
          </div>
          <label :class="{ error: $v.targetInbox.$error }">
            <span v-if="$v.targetInbox.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.INBOX.ERROR') }}
            </span>
          </label>
        </div>
        <div class="w-[50%]">
          <label>
            {{ $t('NEW_CONVERSATION.FORM.TO.LABEL') }}
            <div
              class="flex items-center h-[2.4735rem] rounded-sm py-1 px-2 bg-slate-25 dark:bg-slate-900 border border-solid border-slate-75 dark:border-slate-600"
            >
              <thumbnail
                :src="contact.thumbnail"
                size="24px"
                :username="contact.name"
                :status="contact.availability_status"
              />
              <h4
                class="m-0 ml-2 mr-2 text-slate-700 dark:text-slate-100 text-sm"
              >
                {{ contact.name }}
              </h4>
            </div>
          </label>
        </div>
      </div>
      <div v-if="isAnEmailInbox" class="w-full">
        <div class="w-full">
          <label :class="{ error: $v.subject.$error }">
            {{ $t('NEW_CONVERSATION.FORM.SUBJECT.LABEL') }}
            <input
              v-model="subject"
              type="text"
              :placeholder="$t('NEW_CONVERSATION.FORM.SUBJECT.PLACEHOLDER')"
              @input="$v.subject.$touch"
            />
            <span v-if="$v.subject.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.SUBJECT.ERROR') }}
            </span>
          </label>
        </div>
      </div>
      <div class="w-full">
        <div class="w-full">
          <div class="relative">
            <canned-response
              v-if="showCannedResponseMenu && hasSlashCommand"
              :search-key="cannedResponseSearchKey"
              @click="replaceTextWithCannedResponse"
            />
          </div>
          <div v-if="isEmailOrWebWidgetInbox">
            <label>
              {{ $t('NEW_CONVERSATION.FORM.MESSAGE.LABEL') }}
            </label>
            <reply-email-head
              v-if="isAnEmailInbox"
              :cc-emails.sync="ccEmails"
              :bcc-emails.sync="bccEmails"
            />
            <div class="editor-wrap">
              <woot-message-editor
                v-model="message"
                class="message-editor"
                :class="{ editor_warning: $v.message.$error }"
                :enable-variables="true"
                :signature="signatureToApply"
                :allow-signature="true"
                :placeholder="$t('NEW_CONVERSATION.FORM.MESSAGE.PLACEHOLDER')"
                @toggle-canned-menu="toggleCannedMenu"
                @blur="$v.message.$touch"
              >
                <template #footer>
                  <message-signature-missing-alert
                    v-if="isSignatureEnabledForInbox && !messageSignature"
                    class="!mx-0 mb-1"
                  />
                  <div v-if="isAnEmailInbox" class="mb-3 mt-px">
                    <woot-button
                      v-tooltip.top-end="signatureToggleTooltip"
                      icon="signature"
                      color-scheme="secondary"
                      variant="smooth"
                      size="small"
                      :title="signatureToggleTooltip"
                      @click.prevent="toggleMessageSignature"
                    />
                  </div>
                </template>
              </woot-message-editor>
              <span v-if="$v.message.$error" class="editor-warning__message">
                {{ $t('NEW_CONVERSATION.FORM.MESSAGE.ERROR') }}
              </span>
            </div>
          </div>
          <whatsapp-templates
            v-else-if="hasWhatsappTemplates"
            :inbox-id="selectedInbox.inbox.id"
            @on-select-template="toggleWaTemplate"
            @on-send="onSendWhatsAppReply"
          />
          <label v-else :class="{ error: $v.message.$error }">
            {{ $t('NEW_CONVERSATION.FORM.MESSAGE.LABEL') }}
            <textarea
              v-model="message"
              class="min-h-[5rem]"
              type="text"
              :placeholder="$t('NEW_CONVERSATION.FORM.MESSAGE.PLACEHOLDER')"
              @input="$v.message.$touch"
            />
            <span v-if="$v.message.$error" class="message">
              {{ $t('NEW_CONVERSATION.FORM.MESSAGE.ERROR') }}
            </span>
          </label>
        </div>
      </div>
    </div>
    <div
      v-if="!hasWhatsappTemplates"
      class="flex flex-row justify-end gap-2 py-2 px-0 w-full"
    >
      <button class="button clear" @click.prevent="onCancel">
        {{ $t('NEW_CONVERSATION.FORM.CANCEL') }}
      </button>
      <woot-button type="submit" :is-loading="conversationsUiFlags.isCreating">
        {{ $t('NEW_CONVERSATION.FORM.SUBMIT') }}
      </woot-button>
    </div>
  </form>
</template>

<script>
import { mapGetters } from 'vuex';
import Thumbnail from 'dashboard/components/widgets/Thumbnail.vue';
import WootMessageEditor from 'dashboard/components/widgets/WootWriter/Editor.vue';
import ReplyEmailHead from 'dashboard/components/widgets/conversation/ReplyEmailHead.vue';
import CannedResponse from 'dashboard/components/widgets/conversation/CannedResponse.vue';
import MessageSignatureMissingAlert from 'dashboard/components/widgets/conversation/MessageSignatureMissingAlert';
import InboxDropdownItem from 'dashboard/components/widgets/InboxDropdownItem.vue';
import WhatsappTemplates from './WhatsappTemplates.vue';
import alertMixin from 'shared/mixins/alertMixin';
import { INBOX_TYPES } from 'shared/mixins/inboxMixin';
import { ExceptionWithMessage } from 'shared/helpers/CustomErrors';
import { getInboxSource } from 'dashboard/helper/inbox';
import { required, requiredIf } from 'vuelidate/lib/validators';
import {
  appendSignature,
  removeSignature,
} from 'dashboard/helper/editorHelper';
import uiSettingsMixin from 'dashboard/mixins/uiSettings';

export default {
  components: {
    Thumbnail,
    WootMessageEditor,
    ReplyEmailHead,
    CannedResponse,
    WhatsappTemplates,
    InboxDropdownItem,
    MessageSignatureMissingAlert,
  },
  mixins: [alertMixin, uiSettingsMixin],
  props: {
    contact: {
      type: Object,
      default: () => ({}),
    },
    onSubmit: {
      type: Function,
      default: () => {},
    },
  },
  data() {
    return {
      name: '',
      subject: '',
      message: '',
      showCannedResponseMenu: false,
      cannedResponseSearchKey: '',
      bccEmails: '',
      ccEmails: '',
      targetInbox: {},
      whatsappTemplateSelected: false,
    };
  },
  validations: {
    subject: {
      required: requiredIf('isAnEmailInbox'),
    },
    message: {
      required,
    },
    targetInbox: {
      required,
    },
  },
  computed: {
    ...mapGetters({
      uiFlags: 'contacts/getUIFlags',
      conversationsUiFlags: 'contactConversations/getUIFlags',
      currentUser: 'getCurrentUser',
      messageSignature: 'getMessageSignature',
    }),
    sendWithSignature() {
      const { send_with_signature: isEnabled } = this.uiSettings;
      return isEnabled;
    },
    signatureToApply() {
      return this.messageSignature;
    },
    emailMessagePayload() {
      const payload = {
        inboxId: this.targetInbox.id,
        sourceId: this.targetInbox.sourceId,
        contactId: this.contact.id,
        message: { content: this.message },
        mailSubject: this.subject,
        assigneeId: this.currentUser.id,
      };
      if (this.ccEmails) {
        payload.message.cc_emails = this.ccEmails;
      }

      if (this.bccEmails) {
        payload.message.bcc_emails = this.bccEmails;
      }
      return payload;
    },
    selectedInbox: {
      get() {
        const inboxList = this.contact.contactableInboxes || [];
        return (
          inboxList.find(inbox => {
            return inbox.inbox?.id && inbox.inbox?.id === this.targetInbox?.id;
          }) || {
            inbox: {},
          }
        );
      },
      set(value) {
        this.targetInbox = value.inbox;
      },
    },
    showNoInboxAlert() {
      if (!this.contact.contactableInboxes) {
        return false;
      }
      return this.inboxes.length === 0 && !this.uiFlags.isFetchingInboxes;
    },
    isSignatureEnabledForInbox() {
      return this.isAnEmailInbox && this.sendWithSignature;
    },
    signatureToggleTooltip() {
      return this.sendWithSignature
        ? this.$t('CONVERSATION.FOOTER.DISABLE_SIGN_TOOLTIP')
        : this.$t('CONVERSATION.FOOTER.ENABLE_SIGN_TOOLTIP');
    },
    inboxes() {
      const inboxList = this.contact.contactableInboxes || [];
      return inboxList.map(inbox => ({
        ...inbox.inbox,
        sourceId: inbox.source_id,
      }));
    },
    isAnEmailInbox() {
      return (
        this.selectedInbox &&
        this.selectedInbox.inbox.channel_type === INBOX_TYPES.EMAIL
      );
    },
    isAnWebWidgetInbox() {
      return (
        this.selectedInbox &&
        this.selectedInbox.inbox.channel_type === INBOX_TYPES.WEB
      );
    },
    isEmailOrWebWidgetInbox() {
      return this.isAnEmailInbox || this.isAnWebWidgetInbox;
    },
    hasWhatsappTemplates() {
      return !!this.selectedInbox.inbox?.message_templates;
    },
  },
  watch: {
    message(value) {
      this.hasSlashCommand = value[0] === '/' && !this.isEmailOrWebWidgetInbox;
      const hasNextWord = value.includes(' ');
      const isShortCodeActive = this.hasSlashCommand && !hasNextWord;
      if (isShortCodeActive) {
        this.cannedResponseSearchKey = value.substring(1);
        this.showCannedResponseMenu = true;
      } else {
        this.cannedResponseSearchKey = '';
        this.showCannedResponseMenu = false;
      }
    },
    targetInbox() {
      this.setSignature();
    },
  },
  mounted() {
    this.setSignature();
  },
  methods: {
    setSignature() {
      if (this.messageSignature) {
        if (this.isSignatureEnabledForInbox) {
          this.message = appendSignature(this.message, this.signatureToApply);
        } else {
          this.message = removeSignature(this.message, this.signatureToApply);
        }
      }
    },
    onCancel() {
      this.$emit('cancel');
    },
    onSuccess() {
      this.$emit('success');
    },
    replaceTextWithCannedResponse(message) {
      this.message = message;
    },
    toggleCannedMenu(value) {
      this.showCannedMenu = value;
    },
    prepareWhatsAppMessagePayload({ message: content, templateParams }) {
      const payload = {
        inboxId: this.targetInbox.id,
        sourceId: this.targetInbox.sourceId,
        contactId: this.contact.id,
        message: { content, template_params: templateParams },
        assigneeId: this.currentUser.id,
      };
      return payload;
    },
    onFormSubmit() {
      this.$v.$touch();
      if (this.$v.$invalid) {
        return;
      }
      this.createConversation(this.emailMessagePayload);
    },
    async createConversation(payload) {
      try {
        const data = await this.onSubmit(payload);
        const action = {
          type: 'link',
          to: `/app/accounts/${data.account_id}/conversations/${data.id}`,
          message: this.$t('NEW_CONVERSATION.FORM.GO_TO_CONVERSATION'),
        };
        this.onSuccess();
        this.showAlert(
          this.$t('NEW_CONVERSATION.FORM.SUCCESS_MESSAGE'),
          action
        );
      } catch (error) {
        if (error instanceof ExceptionWithMessage) {
          this.showAlert(error.data);
        } else {
          this.showAlert(this.$t('NEW_CONVERSATION.FORM.ERROR_MESSAGE'));
        }
      }
    },

    toggleWaTemplate(val) {
      this.whatsappTemplateSelected = val;
    },
    async onSendWhatsAppReply(messagePayload) {
      const payload = this.prepareWhatsAppMessagePayload(messagePayload);
      await this.createConversation(payload);
    },
    inboxReadableIdentifier(inbox) {
      return `${inbox.name} (${inbox.channel_type})`;
    },
    computedInboxSource(inbox) {
      if (!inbox.channel_type) return '';
      const classByType = getInboxSource(
        inbox.channel_type,
        inbox.phone_number,
        inbox
      );
      return classByType;
    },
    toggleMessageSignature() {
      this.updateUISettings({
        send_with_signature: !this.sendWithSignature,
      });
      this.setSignature();
    },
  },
};
</script>

<style scoped lang="scss">
.conversation--form {
  @apply pt-4 px-8 pb-8;
}

.message-editor {
  @apply px-3;

  ::v-deep {
    .ProseMirror-menubar {
      @apply rounded-tl-[4px];
    }
  }
}

::v-deep {
  .mention--box {
    @apply left-0 m-auto right-0 top-auto h-fit;
  }
  .multiselect .multiselect__content .multiselect__option span {
    @apply inline-flex w-6 text-slate-600 dark:text-slate-400;
  }
  .multiselect .multiselect__content .multiselect__option {
    @apply py-0.5 px-1;
  }
}
</style>
