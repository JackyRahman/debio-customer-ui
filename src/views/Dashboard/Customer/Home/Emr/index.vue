<template lang="pug">
.customer-emr
  ui-debio-modal(
    :show="showModal"
    :show-title="false"
    @onClose="showModal = false"
  )
    h2.mb-10 Delete
    ui-debio-icon.mb-8(:icon="alertTriangleIcon" stroke size="80")
    p.modal-password__subtitle(v-if="selectedFile") You might not be retrieved back, are you sure you want to delete this EMR ?

    p.modal-password__tx-info.mb-0.d-flex
      span.modal-password__tx-text.mr-6.d-flex.align-center
        | Estimated transaction weight
        ui-debio-icon.ml-1(
          :icon="alertIcon"
          size="14"
          stroke
          @mouseenter="handleShowTooltip"
        )
        span.modal-password__tooltip(
          @mouseleave="handleShowTooltip"
          :class="{ 'modal-password__tooltip--show': showTooltip }"
        ) Total fee paid in DBIO to execute this transaction.
      span.modal-password__tx-value {{ txWeight }}

    .modal-password__cta.d-flex(slot="cta")
      Button(
        block
        color="secondary"
        :loading="isLoading"
        @click="onDelete"
      ) Delete

  ui-debio-banner(
    title="My EMR"
    subtitle="Here, you can upload a collection of your Electronic Medical Records (medical history, diagnoses, medications, treatment plans, immunization dates, allergies, radiology images, and laboratory)."
    gradient-color="primary"
    with-decoration
  )
    template(slot="illustration")
      ui-debio-icon.banner-illustration(:icon="analiticIllustration" :size="cardBlock ? 250 : 180" view-box="0 0 252 252" fill)

  DataTable(
    :headers="headers"
    :loading="isLoading"
    :items="emrDocuments"
  )
    template(slot="prepend")
      .customer-emr__nav
        .customer-emr__nav-text
          h2.customer-emr__title My EMR List
          p.customer-emr__subtitle.mb-0 Your Electronic Medical Records
    template(v-slot:[`item.title`]="{ item }")
      .d-flex.flex-column
        span {{ item.title }}

    template(v-slot:[`item.category`]="{ item }")
      .d-flex.flex-column
        span {{ item.category }}

    template(v-slot:[`item.documentTitle`]="{ item }")
      .d-flex.flex-column
        span.customer-emr__file-title(v-for="(file, idx) in item.files") {{ idx + 1 }}. {{ file.title }}

    template(v-slot:[`item.documentDescription`]="{ item }")
      .d-flex.flex-column
        span.customer-emr__file-description(v-for="(file, idx) in item.files") {{ idx + 1 }}. {{ file.description }}

    template(v-slot:[`item.createdAt`]="{ item }")
      span {{ item.createdAt }}

    template(v-slot:[`item.actions`]="{ item }")
      .customer-emr__actions
        ui-debio-icon(:icon="pencilIcon" size="16" role="button" stroke @click="onEdit(item)")
        ui-debio-icon(:icon="eyeIcon" size="16" role="button" stroke @click="onDetails(item)")
        ui-debio-icon(:icon="trashIcon" size="16" role="button" stroke @click="handleOpenModalDelete(item)")
</template>

<script>
import { mapState } from "vuex"
import {
  layersIcon,
  analiticIllustration,
  eyeIcon,
  pencilIcon,
  eyeOffIcon,
  alertTriangleIcon,
  alertIcon,
  trashIcon,
  downloadIcon
} from "@/common/icons"
import {
  deregisterElectronicMedicalRecord,
  getDeleteEMRFee
} from "@/common/lib/polkadot-provider/command/electronic-medical-record"

import {
  queryGetEMRList,
  queryElectronicMedicalRecordFileById,
  queryElectronicMedicalRecordById
} from "@/common/lib/polkadot-provider/query/electronic-medical-record"
import CryptoJS from "crypto-js"
import Kilt from "@kiltprotocol/sdk-js"
import { u8aToHex } from "@polkadot/util"

import DataTable from "@/common/components/DataTable"
import Button from "@/common/components/Button"
import metamaskServiceHandler from "@/common/lib/metamask/mixins/metamaskServiceHandler"

export default {
  name: "CustomerEmr",
  mixins: [metamaskServiceHandler],

  components: { DataTable, Button },

  data: () => ({
    layersIcon,
    analiticIllustration,
    eyeIcon,
    pencilIcon,
    eyeOffIcon,
    trashIcon,
    downloadIcon,
    alertIcon,
    alertTriangleIcon,

    cardBlock: false,
    showModal: false,
    showModalPassword: false,
    showPassword: false,
    showTooltip: false,
    selectedFile: null,
    error: null,
    txWeight: null,
    publicKey: null,
    secretKey: null,
    headers: [
      {
        text: "EMR Title",
        value: "title",
        sortable: true
      },
      {
        text: "Category",
        value: "category",
        sortable: true
      },
      {
        text: "Document Title",
        value: "documentTitle",
        sortable: true
      },
      {
        text: "Description",
        value: "documentDescription",
        sortable: true
      },
      {
        text: "Upload Date",
        value: "createdAt",
        align: "center",
        sortable: true
      },
      {
        text: "Action",
        value: "actions",
        align: "center"
      }
    ],
    emrDocuments: []
  }),

  computed: {
    ...mapState({
      api: (state) => state.substrate.api,
      wallet: (state) => state.substrate.wallet,
      mnemonicData: (state) => state.substrate.mnemonicData,
      lastEventData: (state) => state.substrate.lastEventData,
      loadingData: (state) => state.auth.loadingData,
      web3: (state) => state.metamask.web3
    })
  },

  watch: {
    lastEventData() {
      if (this.lastEventData !== null) {
        const dataEvent = JSON.parse(this.lastEventData.data.toString())

        if (this.lastEventData.method === "ElectronicMedicalRecordRemoved") {
          if (dataEvent[1] === this.wallet.address) this.getEMRHistory()
        }
      }
    },

    mnemonicData(val) {
      if (val) this.initialDataKey()
    }
  },

  mounted() {
    window.addEventListener("resize", () => {
      if (window.innerWidth <= 959) this.cardBlock = true
      else this.cardBlock = false
    })
  },

  async created() {
    if (this.mnemonicData) this.initialDataKey()
    await this.metamaskDispatchAction(this.getEMRHistory)
  },

  methods: {
    initialDataKey() {
      const cred = Kilt.Identity.buildFromMnemonic(this.mnemonicData.toString(CryptoJS.enc.Utf8))

      this.publicKey = u8aToHex(cred.boxKeyPair.publicKey)
      this.secretKey = u8aToHex(cred.boxKeyPair.secretKey)
    },

    async getEMRHistory() {
      this.emrDocuments = []

      const dataEMR = await this.metamaskDispatchAction(queryGetEMRList, this.api, this.wallet.address)

      if (dataEMR !== null) {
        const listEMR = dataEMR.reduce((filtered, current) => {
          if (filtered.every(v => v !== current)) filtered.push(current)

          return filtered
        }, [])

        if (listEMR.length > 0) {
          listEMR.reverse() // TODO: BAD way, Need reverse from backend
          for (let i = 0; i < listEMR.length; i++) {
            const emrDetail = await this.metamaskDispatchAction(queryElectronicMedicalRecordById,
              this.api,
              listEMR[i]
            )

            if (emrDetail !== null) {
              this.prepareEMRData(emrDetail)
            }
          }

          this.emrDocuments.sort(
            (a, b) => parseInt(b.timestamp) - parseInt(a.timestamp)
          )
        }
      }
    },

    async prepareEMRData(dataEMR) {
      let files = []

      for (let i = 0; i < dataEMR.files?.length; i++) {
        const file = await this.metamaskDispatchAction(queryElectronicMedicalRecordFileById,
          this.api,
          dataEMR.files[i]
        )

        dataEMR.createdAt = new Date(+file.uploadedAt.replaceAll(",", "")).toLocaleDateString("id", {
          day: "2-digit",
          month: "short",
          year: "numeric"
        }),

        files.push({
          ...file,
          createdAt: new Date(+file.uploadedAt.replaceAll(",", "")).toLocaleDateString("id", {
            day: "2-digit",
            month: "short",
            year: "numeric"
          }),
          recordLink: file.recordLink.replace("https://ipfs.io/ipfs/", "")})
      }

      const order = {
        id: dataEMR.id,
        title: dataEMR.title,
        category: dataEMR.category,
        createdAt: dataEMR.createdAt,
        files: files?.slice(0, 3)
      }

      this.emrDocuments.push(order)
    },

    onEdit(emr) {
      this.$router.push({ name: "customer-emr-edit", params: { id: emr.id }})
    },

    onDetails(emr) {
      this.$router.push({ name: "customer-emr-details", params: { id: emr.id }})
    },

    handleShowPassword() {
      this.showPassword = !this.showPassword
    },

    async handleOpenModalDelete(item) {
      this.selectedFile = item
      this.showModal = true

      const txWeight = await getDeleteEMRFee(this.api, this.wallet, item.id)
      this.txWeight = `${this.web3.utils.fromWei(String(txWeight.partialFee), "ether")} DBIO`
    },

    handleShowTooltip(e) {
      if (e.type === "mouseenter") {
        this.showTooltip = true
      } else {
        this.showTooltip = false
      }
    },

    async onDelete() {
      const { id } = this.selectedFile

      try {
        await this.metamaskDispatchAction(deregisterElectronicMedicalRecord, this.api, this.wallet, id)
        this.showModal = false
        this.error = null
        this.selectedFile = null
      } catch (e) {
        this.error = e?.message
      }
    }
  }
}
</script>

<style lang="sass" scoped>
  @import "@/common/styles/mixins.sass"

  .customer-emr
    display: flex
    flex-direction: column
    gap: 30px

    &__actions
      display: flex
      align-items: center
      justify-content: center
      gap: 20px

    &::v-deep
      .banner__subtitle
        max-width: 36.188rem !important

      .ui-debio-modal__card
        gap: 20px

    &__file-title
      max-width: 150px
      white-space: pre
      overflow: hidden
      text-overflow: ellipsis

    &__file-description
      max-width: 150px
      white-space: pre
      overflow: hidden
      text-overflow: ellipsis

    .modal-password
      &__subtitle
        max-width: 280px
        @include body-text-2-opensans

      &__tx-text,
      &__tx-value
        position: relative
        @include body-text-3-opensans

      &__tooltip
        max-width: 143px
        padding: 10px
        position: absolute
        top: 35px
        z-index: 10
        background: #fff
        border: 1px solid #D3C9D1
        right: -120px
        transition: all .3s ease-in-out
        visibility: hidden
        opacity: 0
        @include tiny-reg

        &::after
          position: absolute
          content: ""
          display: block
          top: -20px
          height: 20px
          border-color: transparent transparent #FFF transparent
          border-style: solid
          border-width: 8px

        &::before
          position: absolute
          content: ""
          display: block
          top: -21px
          height: 20px
          border-color: transparent transparent #D3C9D1 transparent
          border-style: solid
          border-width: 8px

        &--show
          opacity: 1
          visibility: visible

      &__cta
        gap: 20px
</style>
