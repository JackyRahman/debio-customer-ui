<template lang="pug">
  .customer-select-service-analyst
    .customer-select-service-analyst__wrapper
      .customer-select-service-analyst__nav
        .customer-select-service-analyst__nav-button(@click="handleBack")
          v-icon.customer-select-service-analyst mdi-chevron-left
          
      .customer-select-service-analyst__main
        .customer-select-service-analyst__stepper
          ui-debio-stepper(
            :items="stepperItems"
          )

        .customer-select-service-analyst__title Select Service and Analyst
        
        .customer-select-service-analyst__cards
          v-row
            v-col(
              v-if="isLoading"
              v-for="(value) in 3"
              :key="value"
            )
              v-skeleton-loader(
                type="card"
                width="300"
              )
            v-col.no-items(
              v-if="!isLoading && !serviceList.length"
            )
              v-img(
                alt="no-list-data"
                center
                src="@/assets/no-data-list.svg"
                width="358px"
                height="207px"
              )
              span.no-items__text-alert Oops! Looks like there is no Service and Analyst available
              span Please try again later
              Button.no-items__button-dashboard(
                color="secondary" 
                width="255"
                height="35"
                @click="goToDashboardPage"
                ) Back to Dashboard
            v-col(
              cols="4"
              v-else
              v-for="(service, i) in serviceList"
              :key="i"
            )
              ServiceAnalysisCard.customer-select-service-analyst__card(
                :service="service"
                @click="showDetail"
              )
              
        AnalystDetail(
          :show="showAnalystDetail"
          :service="selectedService"
          :experiences="selectedAnalystExperiences"
          @close="closeDetailDialog"
        )
        
        ImportantDialog(
          @close="showInformationDialog = false"
          @click="showInformationDialog = false"
          :show="showInformationDialog"
        )
</template>

<script>

import { mapState } from "vuex"
import ServiceAnalysisCard from "./ServiceAnalysisCard"
import AnalystDetail from "./AnalystDetail"
import { queryGeneticAnalysts } from "@/common/lib/polkadot-provider/query/genetic-analysts"
import { queryGetAllGeneticAnalystServices } from "@/common/lib/polkadot-provider/query/genetic-analyst-service"
import { queryGeneticAnalystQualifications } from "@/common/lib/polkadot-provider/query/genetic-analyst-qualifications"
import Button from "@/common/components/Button"
import ImportantDialog from "./Information.vue"

export default {
  name: "SelectServiceAnalyst",

  data: () => ({
    stepperItems: [
      { number: 1, title: "Select Genetic Data", active: false },
      { number: 2, title: "Select Service & Analyst", active: true },
      { number: 3, title: "Checkout and Payment", active: false },
      { number: 4, title: "Success", active: false }
    ],
    showAnalystDetail: false,
    serviceList: [],
    selectedService: null,
    selectedAnalystExperiences: null,
    isLoading: false,
    showInformationDialog: false
  }),

  computed: {
    ...mapState({
      api: (state) => state.substrate.api,
      selectedGeneticData: (state) => state.geneticData.selectedData
    })
  },

  components: {
    ServiceAnalysisCard,
    AnalystDetail,
    Button,
    ImportantDialog
  },

  async mounted () {

    if (!this.selectedGeneticData) {
      this.$router.push({ name: "customer-request-analysis" })
    }

    await this.getGeneticAnalystService()
    this.showInformationDialog = true
  },


  methods: {
    async getGeneticAnalystService() {
      this.isLoading = true
      const geneticAnalystService = await queryGetAllGeneticAnalystServices(this.api)

      for (let i = 0; i < geneticAnalystService.length; i++) {
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
        } = geneticAnalystService[i][1].toHuman()  
        
        const analystsInfo = await queryGeneticAnalysts(this.api, analystId)

        if (analystsInfo.verificationStatus === "Verified") {
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
          this.serviceList.push(service)
        }
      }
      this.isLoading = false
    },

    handleBack() {
      this.$router.push({ name: "customer-request-analysis" })
    },

    async showDetail(val) {
      this.selectedService = val
      const qualifications = val.analystsInfo.qualifications[0]
      this.showAnalystDetail = true
      await this.getExperience(qualifications)
    },

    async getExperience(id) {
      const experiences = await queryGeneticAnalystQualifications(this.api, id)
      this.selectedAnalystExperiences = experiences.info.experience
    },

    closeDetailDialog() {
      this.showAnalystDetail = false
    },

    goToDashboardPage() {
      this.$router.push({name: "dashboard"})
    }
  }


}
</script>

<style lang="sass" scoped>
  @import "@/common/styles/mixins.sass"

  .customer-select-service-analyst
    width: 100%
    height: 100%
    background: #FFFFFF

    &__wrapper
      width: 100%
      height: 100%
      background: #FFFFFF
      box-shadow: 0 0.125rem 0.625rem rgba(0, 0, 0, 0.1)
      border-radius: 0.25rem

    &__nav
      padding: 2.313rem 3.125rem

    &__nav-button
      display: flex
      align-items: center
      gap: 0.625rem
      max-width: max-content
      color: #757274
      cursor: pointer
      @include body-text-3

    &__main
      padding-bottom: 100px
      padding-left: 20px
      padding-right: 32px
      width: 100%
      display: flex
      flex-direction: column
      align-items: center

    &__title
      display: flex
      flex-direction: column
      align-items: center
      margin-top:50px
      @include h6-opensans

    &__cards
      margin-top: 40px
      margin-left: 16px
      
    &__card
      margin-bottom: 20px

  .no-items
    display: flex
    flex-direction: column
    align-items: center
    @include body-text-3-opensans
    font-size: 12px

    &__text-alert
      font-size: 14px
      font-weight: 600
      margin: 25px 0px 20px 0px

    &__button-dashboard
      margin-top: 40px


    
</style>
