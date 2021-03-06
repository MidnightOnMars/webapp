<template>
  <b-modal
    :content-class="darkMode ? 'bg-block-dark' : 'bg-block-light'"
    scrollable
    size="sm"
    centered
    v-model="show"
    hide-footer
    @close="onHide"
    @cancel="onHide"
    @hide="onHide"
    @show="update"
  >
    <template slot="modal-header">
      <div class="w-100">
        <b-row>
          <b-col cols="12" class="d-flex justify-content-between mb-2">
            <span
              class="font-size-14 font-w600"
              :class="darkMode ? 'text-dark' : 'text-light'"
            >
            </span>
            <font-awesome-icon
              class="cursor font-size-lg"
              :class="darkMode ? 'text-dark' : 'text-light'"
              @click="onHide"
              icon="times"
            />
          </b-col>
        </b-row>
      </div>
    </template>

    <div class="text-center" :class="darkMode ? 'text-dark' : 'text-light'">
      <span class="error-icon">
        <font-awesome-icon
          :class="darkMode ? 'text-dark' : 'text-light'"
          icon="times-circle"
        />
      </span>
      <h3
        class="font-size-20 mt-4"
        :class="darkMode ? 'text-dark' : 'text-light'"
      >
        Insufficient {{ symbol }} Balance
      </h3>
      <div class="font-size-16" :class="darkMode ? 'text-dark' : 'text-light'">
        In order to vote, you need to stake {{ symbol }} tokens first. If you do
        not have {{ symbol }}, you will need to stake BNT into a pool first.
      </div>
      <main-button
        @click="onHide"
        label="Stake"
        :large="true"
        :active="true"
        :block="true"
        class="font-size-14 font-w400 mt-3"
      />
    </div>
  </b-modal>
</template>

<script lang="ts">
import { vxm } from "@/store/";
import { Component, Vue } from "vue-property-decorator";
import { VModel } from "@/api/helpers";
import MainButton from "@/components/common/Button.vue";

@Component({
  components: {
    MainButton
  }
})
export default class ModalNotEnoughTokens extends Vue {
  @VModel({ type: Boolean }) show!: boolean;

  symbol: string = "";

  get darkMode(): boolean {
    return vxm.general.darkMode;
  }

  onHide() {
    this.show = false;
  }

  async update() {}

  async mounted() {
    this.symbol = await vxm.ethGovernance.getSymbol();
  }
}
</script>
<style lang="scss">
@import "@/assets/_scss/custom/_variables";

.modal-body {
  padding-top: 0 !important;
}

.error-icon svg {
  font-size: 84px;
  color: $text-error-dark !important;
}
</style>
