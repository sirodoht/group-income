<template lang='pug'>
li.c-item-wrapper(data-test='proposalItem' :data-proposal-hash='proposalHash')
  .c-item
    .c-main
      .c-icons
        i(:class='iconClass')
        avatar.c-avatar(
          v-if='this.proposal.data.proposalData.automated'
          :src='currentGroupState.settings.groupPicture'
        )
        avatar-user.c-avatar(
          v-else
          :contractID='proposal.creatorID' size='xs'
        )

      .c-main-content
        p.c-proposal-title.has-text-bold(data-test='typeDescription')
          | {{typeDescription}}
          tooltip.c-tip(
            v-if='isToRemoveMe && proposal.status === statuses.STATUS_OPEN'
            direction='top'
            :text='L("You cannot vote.")'
          )
            .button.is-icon-smaller.is-primary
              i.icon-question

        p.c-content-title(data-test='title' v-safe-html='subtitle')

        p.has-text-1(
          :class='{ "has-text-danger": proposal.status === statuses.STATUS_FAILED, "has-text-success": proposal.status === statuses.STATUS_PASSED }'
          data-test='statusDescription'
        ) {{statusDescription}}

        .c-reason(v-if='humanReason')
          p.has-text-1.c-reason-text(v-if='humanReason') {{ humanReason }}
          | &nbsp;
          button.link(
            type='button'
            v-if='shouldTruncateReason'
            @click='toggleReason'
          ) {{ ephemeral.isReasonHidden ? L('Read more') : L('Hide') }}

        // Note: $refs.voteMsg is used by children ProposalVoteOptions
        banner-scoped(ref='voteMsg' data-test='voteMsg')
        p.c-sendLink(v-if='invitationLink' data-test='sendLink')
          i18n(
            :args='{ user: proposal.data.proposalData.memberName}'
          ) Please send the following link to {user} so they can join the group:

          link-to-copy.c-invite-link(
            :link='invitationLink'
            tag='p'
          )
          i18n.has-text-danger(
            v-if='isExpiredInvitationLink'
          ) Expired

      proposal-vote-options(
        v-if='proposal.status === statuses.STATUS_OPEN'
        :proposalHash='proposalHash'
      )
</template>

<script>
import { L } from '@common/common.js'
import Avatar from '@components/Avatar.vue'
import AvatarUser from '@components/AvatarUser.vue'
import LinkToCopy from '@components/LinkToCopy.vue'
import Tooltip from '@components/Tooltip.vue'
import BannerScoped from '@components/banners/BannerScoped.vue'
import ProposalVoteOptions from '@containers/proposals/ProposalVoteOptions.vue'
import {
  PROPOSAL_GENERIC,
  PROPOSAL_GROUP_SETTING_CHANGE,
  PROPOSAL_INVITE_MEMBER,
  PROPOSAL_PROPOSAL_SETTING_CHANGE,
  PROPOSAL_REMOVE_MEMBER,
  STATUS_CANCELLED,
  STATUS_EXPIRED,
  STATUS_FAILED,
  STATUS_OPEN,
  STATUS_PASSED
} from '@model/contracts/shared/constants.js'
import currencies from '@model/contracts/shared/currencies.js'
import { humanDate } from '@model/contracts/shared/time.js'
import { RULE_DISAGREEMENT, RULE_PERCENTAGE, VOTE_AGAINST, VOTE_FOR, getPercentFromDecimal } from '@model/contracts/shared/voting/rules.js'
import { buildInvitationUrl } from '@view-utils/buildInvitationUrl.js'
import { TABLET } from '@view-utils/breakpoints.js'
import { mapGetters, mapState } from 'vuex'
import { INVITE_STATUS } from '~/shared/domains/chelonia/constants.js'

export default ({
  name: 'ProposalItem',
  props: {
    proposalHash: String,
    proposalObject: Object
  },
  data () {
    return {
      config: {
        reasonMaxLength: window.innerWidth < TABLET ? 50 : 170
      },
      ephemeral: {
        isReasonHidden: true
      }
    }
  },
  components: {
    BannerScoped,
    ProposalVoteOptions,
    AvatarUser,
    Avatar,
    LinkToCopy,
    Tooltip
  },
  computed: {
    ...mapGetters([
      'currentGroupState',
      'groupMembersCount',
      'userDisplayNameFromID',
      'ourIdentityContractId',
      'ourUserDisplayName'
    ]),
    ...mapState(['currentGroupId']),
    statuses () {
      return { STATUS_OPEN, STATUS_PASSED, STATUS_FAILED, STATUS_EXPIRED, STATUS_CANCELLED }
    },
    proposal () {
      return this.proposalObject || this.currentGroupState.proposals[this.proposalHash]
    },
    subtitle () {
      const creatorID = this.proposal.creatorID
      const username = this.userDisplayNameFromID(creatorID)
      const isOwnProposal = creatorID === this.ourIdentityContractId

      if (this.proposal.data.proposalData.automated) {
        return L('Group Income system is proposing')
      }
      if (this.proposal.status === STATUS_OPEN) {
        return isOwnProposal
          ? L('You are proposing')
          : L('{username} is proposing', { username })
      }

      // Note: In English, no matter the subject, the wording is the same,
      // but in other languages the wording is different (ex: Portuguese)
      return isOwnProposal
        ? L('You proposed')
        : L('{username} proposed', { username })
    },
    proposalType () {
      return this.proposal.data.proposalType
    },
    isOurProposal () {
      return this.proposal.creatorID === this.ourIdentityContractId
    },
    isToRemoveMe () {
      return this.proposalType === PROPOSAL_REMOVE_MEMBER && this.proposal.data.proposalData.memberID === this.ourIdentityContractId
    },
    typeDescription () {
      return {
        [PROPOSAL_INVITE_MEMBER]: () => L('Add {user} to group.', {
          user: this.proposal.data.proposalData.memberName
        }),
        [PROPOSAL_REMOVE_MEMBER]: () => {
          const user = this.userDisplayNameFromID(this.proposal.data.proposalData.memberID)
          const automated = this.proposal.data.proposalData.automated ? `[${L('Automated')}] ` : ''
          return this.isToRemoveMe
            ? L('{automated}Remove {user} (you) from the group.', { user, automated })
            : L('{automated}Remove {user} from the group.', { user, automated })
        },
        [PROPOSAL_GROUP_SETTING_CHANGE]: () => {
          const { setting } = this.proposal.data.proposalData
          // TODO layout for this type of proposal. Waiting for designs.
          const variablesMap = {
            'mincomeAmount': () => {
              const { mincomeCurrency, currentValue, proposedValue } = this.proposal.data.proposalData

              return {
                setting: L('mincome'),
                currentValue: currencies[mincomeCurrency].displayWithCurrency(currentValue),
                proposedValue: currencies[mincomeCurrency].displayWithCurrency(proposedValue)
              }
            },
            'distributionDate': () => {
              const { currentValue, proposedValue } = this.proposal.data.proposalData

              return {
                setting: L('distribution date'),
                currentValue: humanDate(currentValue, { month: 'long', year: 'numeric', day: 'numeric' }),
                proposedValue: humanDate(proposedValue, { month: 'long', year: 'numeric', day: 'numeric' })
              }
            }
          }[setting]()

          return L('Change {setting} from {currentValue} to {proposedValue}', variablesMap)
        },
        [PROPOSAL_PROPOSAL_SETTING_CHANGE]: () => {
          const { current, ruleName, ruleThreshold } = this.proposal.data.proposalData

          if (current.ruleName === ruleName) {
            return {
              [RULE_DISAGREEMENT]: () => L('Change disagreement number from {X} to {N}.', {
                X: current.ruleThreshold,
                N: ruleThreshold
              }),
              [RULE_PERCENTAGE]: () => L('Change percentage based from {X} to {N}.', {
                X: getPercentFromDecimal(current.ruleThreshold) + '%',
                N: getPercentFromDecimal(ruleThreshold) + '%'
              })
            }[ruleName]()
          }

          return {
            [RULE_DISAGREEMENT]: () => L('Change from a percentage based voting system to a disagreement based one, with a maximum of {N} “no” votes.', {
              N: ruleThreshold
            }),
            [RULE_PERCENTAGE]: () => L('Change from a disagreement based voting system to a percentage based one, with minimum agreement of {percent}.', {
              percent: getPercentFromDecimal(ruleThreshold) + '%'
            })
          }[ruleName]()
        },
        [PROPOSAL_GENERIC]: () => this.proposal.data.proposalData.name
      }[this.proposalType]()
    },
    statusDescription () {
      const votes = Object.values(this.proposal.votes)
      const format = { year: 'numeric', month: 'long', day: 'numeric' }
      const yay = votes.filter(v => v === VOTE_FOR).length
      const nay = votes.filter(v => v === VOTE_AGAINST).length
      const total = yay + nay
      switch (this.proposal.status) {
        case STATUS_OPEN: {
          const excludeVotes = this.proposalType === PROPOSAL_REMOVE_MEMBER ? 1 : 0
          const date = humanDate(this.proposal.data.expires_date_ms, format)
          return L('{count} out of {total} members voted. Expires on {date}.', {
            count: votes.length,
            total: this.groupMembersCount - excludeVotes,
            date
          })
        }
        case STATUS_FAILED: {
          const date = humanDate(this.proposal.dateClosed, format)
          return L('Proposal rejected on {date} with {nay} against out of {total} total votes.', { nay, total, date })
        }
        case STATUS_CANCELLED: {
          return L('Proposal cancelled.')
        }
        case STATUS_PASSED: {
          const date = humanDate(this.proposal.dateClosed, format)
          return L('Proposal accepted on {date} with {yay} in favor out of {total} total votes.', { yay, total, date })
        }
        case STATUS_EXPIRED: {
          const date = humanDate(this.proposal.data.expires_date_ms, format)
          return L('Expired on {date}', { date })
        }
        default:
          return `status: ${this.proposal.status}`
      }
    },
    iconClass () {
      const type = {
        [PROPOSAL_INVITE_MEMBER]: 'icon-user-plus',
        [PROPOSAL_REMOVE_MEMBER]: 'icon-user-minus',
        [PROPOSAL_GROUP_SETTING_CHANGE]: 'icon-coins',
        [PROPOSAL_PROPOSAL_SETTING_CHANGE]: 'icon-vote-yea',
        [PROPOSAL_GENERIC]: 'icon-envelope-open-text'
      }

      const status = {
        [STATUS_OPEN]: 'has-background-primary has-text-primary',
        [STATUS_PASSED]: 'icon-check has-background-success has-text-success',
        [STATUS_FAILED]: 'icon-times has-background-danger has-text-danger',
        [STATUS_CANCELLED]: 'has-background-general has-text-1',
        [STATUS_EXPIRED]: 'has-background-general has-text-1'
      }

      if ([STATUS_PASSED, STATUS_FAILED].includes(this.proposal.status)) {
        // Show the status icon, no matter the proposal type
        return `${status[this.proposal.status]} icon-round`
      }

      return `${type[this.proposalType]} ${status[this.proposal.status]} icon-round`
    },
    shouldTruncateReason () {
      const reason = this.proposal.data.proposalData.reason
      const threshold = 40 // avoid clicking "read more" and see only a few more characters.
      return reason.length > this.config.reasonMaxLength + threshold
    },
    humanReason () {
      const reason = this.proposal.data.proposalData.reason
      const maxlength = this.config.reasonMaxLength
      if (this.ephemeral.isReasonHidden && this.shouldTruncateReason) {
        // Prevent "..." to be added after an empty space. ex: "they would ..." -> "they would..."
        const charToTruncate = reason.charAt(maxlength - 1) === ' ' ? maxlength - 1 : maxlength
        return `"${reason.substr(0, charToTruncate)}..."`
      }

      return reason ? `"${reason}"` : ''
    },
    invitationLink () {
      if (this.proposalType === PROPOSAL_INVITE_MEMBER &&
        this.proposal.status === STATUS_PASSED &&
        this.isOurProposal
      ) {
        const inviteKeyId = this.proposal.payload.inviteKeyId
        // Display the link for (1) valid invites for which (2) there is a
        // corresponding authorizedKey for which (3) we have access to its
        // secret key
        if (
          this.currentGroupState._vm.invites?.[inviteKeyId]?.status === INVITE_STATUS.VALID &&
          this.currentGroupState._vm.authorizedKeys?.[inviteKeyId] &&
          this.currentGroupState._vm.authorizedKeys[inviteKeyId]._notAfterHeight == null &&
          this.currentGroupState._vm.invites[inviteKeyId].inviteSecret
        ) {
          return buildInvitationUrl(this.currentGroupId, this.currentGroupState.settings?.groupName, this.currentGroupState._vm.invites[inviteKeyId].inviteSecret, this.ourIdentityContractId)
        }
      }
      return false
    },
    isExpiredInvitationLink () {
      const inviteKeyId = this.proposal.payload.inviteKeyId
      if (
        this.currentGroupState._vm.invites[inviteKeyId]?.status !== INVITE_STATUS.VALID ||
        // inviteKeyId should be present in authorizedKeys. If it's not, it's
        // an error but it also means that the invite cannot be used
        !this.currentGroupState._vm?.authorizedKeys?.[inviteKeyId] ||
        // If _notAfterHeight is *not* undefined, it means that the key has been
        // revoked. Hence, it cannot be used
        this.currentGroupState._vm.authorizedKeys[inviteKeyId]._notAfterHeight !== undefined ||
        // If the expiration date is less than the current date, it means that
        // the invite can no longer be used
        // Note: Using negative logic to allow for undefined expiry, which means
        // it never expires
        this.currentGroupState._vm.invites[inviteKeyId].expires < Date.now()
      ) {
        return true
      }
      return false
    }
  },
  methods: {
    toggleReason (e) {
      e.target.blur() // so the button doesnt remain focused (with black color).
      this.ephemeral.isReasonHidden = !this.ephemeral.isReasonHidden
    }
  }
}: Object)
</script>

<style lang="scss" scoped>
@import "@assets/style/_variables.scss";

.c-item-wrapper {
  margin-top: 2rem;

  &:not(:last-child) {
    padding-bottom: 2rem;
    border-bottom: 1px solid $general_1;
  }
}

.c-item {
  display: flex;
  align-items: flex-start;

  @include phone {
    flex-wrap: wrap;
  }
}

.c-main {
  grid-template-columns: auto 1fr auto;
  display: grid;
  grid-template-areas: "icons content actions";
  width: 100%;

  @include phone {
    grid-template-areas: "icons content content" "icons actions actions";
  }

  &-content {
    flex-grow: 1;
  }

  .c-content-title {
    word-break: break-word;
  }
}

.c-tip {
  margin-left: 0.25rem;
}

.c-sendLink {
  border-radius: 0.25rem;
  background-color: $general_2;
  padding: 1.1875rem 1rem;
  margin-top: 1rem;
  display: grid;

  @include tablet {
    padding: 1rem;
  }

  .c-invite-link {
    ::v-deep .c-copy-button {
      background: $background_0;
    }
  }
}

.icon-round {
  @include phone {
    margin-left: 0.5rem;
  }
}

.c-icons {
  position: relative;
  align-self: flex-start;
  grid-area: icons;

  .icon-round {
    width: 3.75rem;
    height: 3.75rem;
    display: grid;
    align-items: center;

    @include phone {
      width: 2.75rem;
      height: 2.75rem;
    }
  }

  .c-avatar {
    position: absolute;
    bottom: -0.2rem;
    right: 1rem;

    @include phone {
      display: none;
    }
  }
}

.c-reason {
  position: relative;
  margin-top: 1rem;

  &-text {
    display: inline;
  }
}

.c-proposal-title,
.c-reason {
  word-break: break-word;
}
</style>
