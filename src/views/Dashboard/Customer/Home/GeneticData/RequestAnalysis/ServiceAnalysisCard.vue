<template lang="pug">
  v-card.service-analysis-card(@click="onClick")
    .service-analysis-card__title {{ serviceName }}

    .service-analysis-card__description {{ computeDescription }}

    .service-analysis-card__info
      v-row
        v-col(cols="8")
          v-icon(size="14" outlined ) mdi-timer 
          span.service-analysis-card__info-duration {{ duration }} {{ durationType }}
        v-col(cols="4")
          b.service-analysis-card__info-price {{ price }}

    hr

    .service-analysis-card__analyst
      v-row
        v-col(cols=3)
          ui-debio-avatar.service-analysis-card__avatar(
            :src="computeAvatar" 
            size="75"
            rounded
          )
  

        v-col(cols=8).service-analysis-card__analyst-info
          .service-analysis-card__analyst-name {{ analystName }}
          .service-analysis-card__analyst-desc {{ specialization }}

          a(:href="profileLink" target="_blank")
            v-img.service-analysis-card__social(
              alt="linkedin"
              center
              src="@/assets/linkedin-logo.png"
              height="15" 
              width="15"
            )

</template>

<script>
import { mapState } from "vuex"
import { queryGeneticAnalysisOrders } from "@/common/lib/polkadot-provider/query/genetic-analysis-orders"
import { queryGetGeneticAnalystServiceById } from "@/common/lib/polkadot-provider/query/genetic-analyst-service"
import { queryGeneticAnalysts } from "@/common/lib/polkadot-provider/query/genetic-analysts"


export default {
  name: "ServiceAnalysisCard",

  data: () => ({
    showDetail: false,
    serviceName: null,
    description: null,
    duration: null,
    durationType: null,
    price: null,
    profileImage: null,
    analystName: null,
    specialization: null,
    profileLink: null,
    geneticOrderDetail: null,
    orderId: null
  }),

  props: {
    service: Object
  },

  computed: {
    ...mapState({
      api: (state) => state.substrate.api,
      web3: (state) => state.metamask.web3
    }),

    computeDescription() {
      const validateLength = this.description.length >= 100
        ? `${this.description.slice(0, 100)}...`
        : this.description

      return this.description ? validateLength : "No Description"
    },

    computeAvatar() {
      return this.profileImage ? this.profileImage : require("@/assets/defaultAvatar.svg")
    }
  },

  async mounted() {

    if(this.$route.params.id) {
      this.isCreated = true
      this.orderId = this.$route.params.id
      await this.getServiceDetail()
    }

    this.serviceName = this.service.serviceName
    this.description = this.service.description
    this.duration = this.service.duration
    this.durationType = this.service.durationType
    this.price = `${this.formatBalance(this.service.priceDetail[0].totalPrice)} ${this.service.priceDetail[0].currency}`
    this.analystName = `${this.service.analystsInfo.info.firstName} ${this.service.analystsInfo.info.lastName}`
    this.specialization = this.service.analystsInfo.info.specialization
    this.profileImage = this.service.analystsInfo.info.profileImage
    this.profileLink = this.service.analystsInfo.info.profileLink
  },

  methods: {
    onClick() {
      this.$emit("click", this.service)
    },

    formatBalance(balance) {
      const formatedBalance = this.web3.utils.fromWei(String(balance.replaceAll(",", "")), "ether")
      return Number(formatedBalance).toPrecision()
    },

    async getServiceDetail ()  {
      this.geneticOrderDetail = await queryGeneticAnalysisOrders(this.api, this.orderId)
      
      const serviceDetail = await queryGetGeneticAnalystServiceById(this.api, this.geneticOrderDetail.serviceId)

      let {
        id: serviceId,
        ownerId: analystId,
        info: {
          name: serviceName,
          pricesByCurrency: priceDetail,
          expectedDuration: {
            duration,
            durationType
          },
          description,
          testResultSample
        }     
      } = serviceDetail

      const analystsInfo = await queryGeneticAnalysts(this.api, analystId)
      const service = {
        serviceId,
        analystId,
        serviceName,
        priceDetail,
        duration,
        durationType,
        description,
        testResultSample,
        analystsInfo
      }

      this.serviceName = service.serviceName
      this.description = service.description
      this.duration = service.duration
      this.durationType = service.durationType
      this.price = `${this.formatBalance(service.priceDetail[0].totalPrice)} ${service.priceDetail[0].currency}`
      this.analystName = `${service.analystsInfo.info.firstName} ${service.analystsInfo.info.lastName}`
      this.specialization = service.analystsInfo.info.specialization
      this.profileImage = service.analystsInfo.info.profileImage
      this.profileLink = service.analystsInfo.info.profileLink

    }
  }
}
</script>

<style lang="sass" scoped>
  @import "@/common/styles/mixins.sass"

  .service-analysis-card
    margin-top: 0 !important
    width: 340px
    height: 320px
    cursor: pointer

    &__title
      padding-top: 25px
      margin-left: 20px
      letter-spacing: -0.0075em
      @include button-2

    &__description
      height: 83px
      margin: 20px
      letter-spacing: -0.0044em
      @include new-body-text-2

    &__info
      margin-left: 16px
      margin-bottom: 0 !important
      padding-bottom: 20px

    &__info-duration
      letter-spacing: -0.004em
      margin-left: 5px
      @include body-text-3-opensans-medium

    &__info-price
      color: #F006CB
      @include body-text-3-opensans

    &__avatar
      margin-top: 16px
      margin-left: 16px

    &__img
      margin-top: 16px
      margin-left: 12px
      margin-right: -10px
      
    &__analyst
      margin-bottom: 20px

    &__analyst-info
      margin-left: 8px

    &__analyst-name
      margin-top: 16px
      @include button-2

    &__analyst-desc
      @include body-text-3-opensans

    &__social
      margin-top: 16px
</style>
