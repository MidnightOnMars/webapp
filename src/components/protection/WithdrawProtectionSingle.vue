<template>
  <div class="mt-3">
    <label-content-split label="Initial Stake">
      <logo-amount-symbol
        :pool-id="position.stake.poolId"
        :amount="prettifyNumber(position.stake.amount)"
        :symbol="position.stake.symbol"
      />
    </label-content-split>

    <label-content-split
      label="Fully Protected Value"
      :value="
        `${prettifyNumber(position.protectedAmount.amount)} ${
          position.stake.symbol
        }`
      "
      class="my-3"
    />

    <alert-block
      v-if="warning"
      variant="warning"
      title="Important"
      :msg="warning"
      class="my-3"
    />

    <percentage-slider
      label="Input"
      v-model="percentage"
      @input="onPercentUpdate"
      :show-buttons="true"
    />

    <div class="d-flex justify-content-center mb-3">
      <font-awesome-icon icon="arrow-down" class="mt-3" />
    </div>

    <gray-border-block :gray-bg="true" class="my-3">
      <label-content-split
        v-if="expectedValue"
        label="Output value of"
        :value="
          `${prettifyNumber(expectedValue.amount)} ${expectedValue.symbol}`
        "
      />

      <label-content-split
        v-for="(output, index) in outputs"
        :label="index == 0 ? 'Output breakdown' : ''"
        :key="output.id"
        :value="`${prettifyNumber(output.amount)} ${output.symbol}`"
      >
      </label-content-split>
    </gray-border-block>

    <alert-block
      v-if="vBntWarning"
      variant="error"
      :msg="vBntWarning"
      class="my-3"
    />

    <main-button
      label="Continue"
      @click="initAction"
      :active="true"
      :large="true"
      :disabled="disableActionButton"
    />

    <modal-base title="You will receive" v-model="modal" @input="setDefault">
      <action-modal-status :error="error" :success="success" />

      <main-button
        @click="onModalClick"
        class="mt-3"
        :label="modalConfirmButton"
        :active="true"
        :large="true"
        :disabled="txBusy"
      />
    </modal-base>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";
import { vxm } from "@/store/";
import {
  TxResponse,
  ViewAmount,
  ViewAmountDetail,
  ViewRelay
} from "@/types/bancor";
import GrayBorderBlock from "@/components/common/GrayBorderBlock.vue";
import LabelContentSplit from "@/components/common/LabelContentSplit.vue";
import MainButton from "@/components/common/Button.vue";
import PercentageSlider from "@/components/common/PercentageSlider.vue";
import AlertBlock from "@/components/common/AlertBlock.vue";
import { compareString, findOrThrow, prettifyNumber } from "@/api/helpers";
import ModalBase from "@/components/modals/ModalBase.vue";
import ActionModalStatus from "@/components/common/ActionModalStatus.vue";
import LogoAmountSymbol from "@/components/common/LogoAmountSymbol.vue";
import BigNumber from "bignumber.js";

interface ViewAmountUsd extends ViewAmount {
  usdValue: number;
}

@Component({
  components: {
    LogoAmountSymbol,
    ActionModalStatus,
    ModalBase,
    AlertBlock,
    PercentageSlider,
    LabelContentSplit,
    GrayBorderBlock,
    MainButton
  }
})
export default class WithdrawProtectionSingle extends Vue {
  @Prop() pool!: ViewRelay;

  percentage: string = "50";

  modal = false;
  txBusy = false;
  success: TxResponse | string | null = null;
  error = "";
  outputs: ViewAmountDetail[] = [];
  expectedValue: ViewAmountDetail | null = null;

  get warning() {
    return this.position.whitelisted && this.position.coverageDecPercent !== 1
      ? "You still haven’t reached full coverage. There is a risk for impermanent loss."
      : "";
  }

  get disableActionButton() {
    if (this.vBntWarning) return true;
    else if (parseFloat(this.percentage) === 0) return true;
    else return this.inputError ? true : false;
  }

  get inputError() {
    if (parseFloat(this.percentage) === 0) return "Percentage can not be Zero";
    else return "";
  }

  get position() {
    const [poolId, first, second] = this.$route.params.id.split(":");

    const pos = findOrThrow(vxm.ethBancor.protectedPositions, position =>
      compareString(position.id, this.$route.params.id)
    );
    console.log(pos, "is the selected pos");
    return pos;
  }

  get vBntWarning() {
    return this.position.givenVBnt && !this.sufficientVBnt
      ? `Insufficient vBNT balance, you must hold ${prettifyNumber(
          Number(this.position.givenVBnt!) * (Number(this.percentage) / 100)
        )} vBNT before withdrawing position.`
      : "";
  }

  get sufficientVBnt() {
    if (this.position.givenVBnt) {
      const decPercent = new BigNumber(this.percentage).div(100);
      const proposedWithdraw = new BigNumber(this.position.givenVBnt).times(
        decPercent
      );
      return proposedWithdraw.isLessThanOrEqualTo(this.vBntBalance);
    } else return true;
  }

  get vBntBalance() {
    const balance = vxm.ethBancor.tokenBalance(
      vxm.ethBancor.liquidityProtectionSettings.govToken
    );
    return balance ? balance.balance : "0";
  }

  async initAction() {
    this.setDefault();
    this.modal = true;
    this.txBusy = true;
    const [poolId, first, second] = this.$route.params.id.split(":");
    console.log({ poolId, first, second });
    try {
      const txRes = await vxm.ethBancor.removeProtection({
        decPercent: Number(this.percentage) / 100,
        id: this.position.id
      });
      this.success = txRes;
    } catch (err) {
      this.error = err.message;
    } finally {
      this.txBusy = false;
    }
  }

  onModalClick() {
    if (this.success) {
      this.setDefault();
      this.modal = false;
      this.$router.push({ name: "LiqProtection" });
    } else if (this.error) {
      this.setDefault();
      this.modal = false;
    }
  }

  prettifyNumber(amount: string | number, usd = false) {
    return prettifyNumber(amount, usd);
  }

  setDefault() {
    this.error = "";
    this.success = null;
    this.txBusy = false;
  }

  async onPercentUpdate(newPercent: string) {
    console.log(newPercent, "is the new percent");
    const res = await vxm.ethBancor.calculateSingleWithdraw({
      id: this.position.id,
      decPercent: Number(this.percentage) / 100
    });

    this.expectedValue = res.expectedValue;
    this.outputs = res.outputs;

    console.log(res, "was the res");
  }

  created() {
    this.onPercentUpdate(this.percentage);
  }

  get modalConfirmButton() {
    return this.error
      ? "Close"
      : this.success
      ? "Close"
      : this.txBusy
      ? "processing ..."
      : "Confirm";
  }

  get darkMode() {
    return vxm.general.darkMode;
  }
}
</script>

<style scoped lang="scss"></style>
