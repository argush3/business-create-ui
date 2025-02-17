<template>
  <div id="association-details">
    <article class="section-container">
      <!-- Entity Name, etc -->
      <v-row no-gutters>
        <v-col
          cols="12"
          sm="3"
          class="pr-4"
        >
          <label>{{ entityLabel }}</label>
        </v-col>

        <v-col
          cols="12"
          sm="9"
          class="mt-n1"
        >
          <label id="company-name">{{ entityName }}</label>
          <div class="my-1">
            {{ entityDescription }}
          </div>
          <div>{{ getBusinessId }}</div>
        </v-col>
      </v-row>
    </article>

    <v-divider class="mx-6" />

    <!-- Address -->
    <article class="section-container">
      <v-row no-gutters>
        <v-col
          cols="12"
          sm="3"
          class="pr-4"
        >
          <label>{{ addressLabel }}</label>
        </v-col>

        <v-col
          cols="12"
          sm="4"
          class="pr-4 pt-4 pt-sm-0"
        >
          <label class="mailing-address-header">Mailing Address</label>
          <MailingAddress
            v-if="!isEmptyAddress(getBusinessOfficeAddress.mailingAddress)"
            :address="getBusinessOfficeAddress.mailingAddress"
            :editing="false"
          />
          <div v-else>
            (Not entered)
          </div>
        </v-col>

        <v-col
          cols="12"
          sm="4"
          class="pr-4 pt-4 pt-sm-0"
        >
          <label class="delivery-address-header">Delivery Address</label>
          <DeliveryAddress
            v-if="!isEmptyAddress(getBusinessOfficeAddress.deliveryAddress) &&
              !isSame(getBusinessOfficeAddress.mailingAddress, getBusinessOfficeAddress.deliveryAddress, ['id'])"
            :address="getBusinessOfficeAddress.deliveryAddress"
            :editing="false"
          />
          <div v-else-if="isEmptyAddress(getBusinessOfficeAddress.deliveryAddress)">
            (Not entered)
          </div>
          <div v-else>
            Same as Mailing Address
          </div>
        </v-col>
      </v-row>
    </article>

    <template v-if="showBusinessDate">
      <v-divider class="mx-6" />
      <!-- Business Start Date -->
      <article class="section-container">
        <v-row no-gutters>
          <v-col
            cols="12"
            sm="3"
            class="pr-4"
          >
            <label>Business Start Date</label>
          </v-col>
          <v-col
            cols="12"
            sm="9"
          >
            <div>{{ businessStartDate || '(Not entered)' }}</div>
          </v-col>
        </v-row>
      </article>
    </template>

    <!-- Folio Number -->
    <template v-if="isPremiumAccount">
      <v-divider class="mx-6" />

      <article class="section-container">
        <v-row no-gutters>
          <v-col
            cols="12"
            sm="3"
            class="pr-4"
          >
            <label>Folio or Reference Number</label>
          </v-col>

          <v-col
            cols="12"
            sm="9"
          >
            <div id="lbl-folio-number">
              {{ getFolioNumber || '(Not entered)' }}
            </div>
          </v-col>
        </v-row>
      </article>
    </template>

    <template v-if="showContactInfo">
      <v-divider class="mx-6" />

      <!-- Contact Info -->
      <article class="section-container">
        <ContactInfo
          :businessContact="getBusinessContact"
          :disableActions="isSummary"
          :customMsg="contactInfoMsg"
          editLabel="Change"
          @contactInfoChange="onContactInfoChange($event)"
        />
      </article>
    </template>
  </div>
</template>

<script lang="ts">
import { Component, Mixins, Prop } from 'vue-property-decorator'
import { Action, Getter } from 'pinia-class'
import { useStore } from '@/store/store'
import { AuthServices } from '@/services/'
import { ActionBindingIF, AddressIF, ContactPointIF, OfficeAddressIF } from '@/interfaces'
import { ContactInfo } from '@bcrs-shared-components/contact-info'
import BaseAddress from 'sbc-common-components/src/components/BaseAddress.vue'
import OfficeAddresses from '@/components/common/OfficeAddresses.vue'
import { CommonMixin, DateMixin } from '@/mixins'
import { CoopTypes } from '@/enums'
import { CorpTypeCd } from '@bcrs-shared-components/enums/'
import { isEmpty } from 'lodash'
import { GetCorpFullDescription, GetCorpNumberedDescription } from '@bcrs-shared-components/corp-type-module'

@Component({
  components: {
    ContactInfo,
    OfficeAddresses,
    DeliveryAddress: BaseAddress,
    MailingAddress: BaseAddress
  }
})
export default class AssociationDetails extends Mixins(CommonMixin, DateMixin) {
  @Prop({ default: false }) readonly isSummary!: boolean
  @Prop({ default: 'Address' }) readonly addressLabel!: string
  @Prop({ default: 'Company' }) readonly entityLabel!: string
  @Prop({ default: false }) readonly showBusinessDate!: boolean
  @Prop({ default: true }) readonly showContactInfo!: boolean

  // Global getters
  @Getter(useStore) getBusinessContact!: ContactPointIF
  @Getter(useStore) getBusinessFoundingDate!: string
  @Getter(useStore) getBusinessId!: string
  @Getter(useStore) getBusinessLegalName!: string
  @Getter(useStore) getBusinessOfficeAddress!: OfficeAddressIF
  @Getter(useStore) getCompanyDisplayName!: string
  @Getter(useStore) getCooperativeType!: CoopTypes
  @Getter(useStore) getEntityType!: CorpTypeCd
  @Getter(useStore) getFolioNumber!: string
  @Getter(useStore) isPremiumAccount!: boolean
  @Getter(useStore) isTypeCoop!: boolean

  // Global setters
  @Action(useStore) setBusinessContact!: ActionBindingIF
  @Action(useStore) setIgnoreChanges!: ActionBindingIF

  readonly contactInfoMsg = `Registered Office Contact Information is required for dissolution documents delivery.
  Any changes made will be applied immediately.`

  /** The entity name. */
  get entityName (): string {
    return this.getBusinessLegalName || GetCorpNumberedDescription(this.getEntityType)
  }

  /** The entity description.  */
  get entityDescription (): string {
    return GetCorpFullDescription(this.getEntityType)
  }

  /** The business start date. */
  get businessStartDate (): string {
    // it will be same as foundingDate
    return this.dateToPacificDate(this.apiToDate(this.getBusinessFoundingDate), true)
  }

  /** Whether the address object is empty. */
  isEmptyAddress (address: AddressIF): boolean {
    return isEmpty(address)
  }

  /** Event handler for contact information changes. */
  async onContactInfoChange (event: ContactPointIF): Promise<void> {
    // temporarily ignore data changes
    this.setIgnoreChanges(true)

    await AuthServices.updateContactInfo(this.getBusinessId, event).then(response => {
      this.setBusinessContact(response)
    }).catch(error => {
      this.$root.$emit('save-error-event', error)
    })

    // watch further changes
    this.setIgnoreChanges(false)
  }
}
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

.mailing-address-header,
.delivery-address-header {
  font-size: $px-14;
}

#company-name {
  font-size: $px-22;
  color: $gray9;
}
</style>
