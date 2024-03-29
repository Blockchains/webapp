<template>
  <base-modal
    id="modal-pool-action"
    v-on:on-hide-modal="setDefault"
    title="You will receive"
  >
    <b-row class="d-flex justify-content-center">
      <div v-if="!(txBusy || success || error)" class="w-100">
        <b-col
          v-if="false"
          cols="12"
          class="d-flex align-items-center justify-content-center"
        >
          <span
            class="font-size-24 font-w600 mr-2"
            :class="darkMode ? 'text-dark' : 'text-light'"
          >
            {{ amountsArray[0] }}
          </span>
          <pool-logos :pool="pool" />
        </b-col>

        <b-col v-else cols="12" class="text-center">
          <div
            v-if="selectedToken && pool.v2"
            class="font-size-24 font-w600 mr-2"
            :class="darkMode ? 'text-dark' : 'text-light'"
          >
            {{ amountsArray[1] }} {{ selectedToken.symbol }}
          </div>
          <div v-else>
            <div
              class="font-size-24 font-w600 mr-2"
              :class="darkMode ? 'text-dark' : 'text-light'"
            >
              {{ amountsArray[1] }} {{ pool.reserves[0].symbol }}
            </div>
            <font-awesome-icon
              v-if="!pool.v2"
              icon="plus"
              class="text-primary"
            />
            <div
              v-if="!pool.v2"
              class="font-size-24 font-w600 mr-2"
              :class="darkMode ? 'text-dark' : 'text-light'"
            >
              {{ amountsArray[2] }} {{ pool.reserves[1].symbol }}
            </div>
          </div>
        </b-col>

        <b-col cols="12" v-if="pool.v2">
          <p
            class="font-size-sm font-w400 text-center mt-2"
            :class="!darkMode ? 'text-muted-light' : 'text-muted-dark'"
          >
            Output is estimated. If the price changes by more than
            {{ numeral(slippageTolerance).format("0.0[0]%") }} your transaction
            will revert.
          </p>
        </b-col>

        <b-col cols="12">
          <div
            class="block block-rounded font-size-sm block-shadow my-3"
            :class="darkMode ? 'bg-body-dark' : 'bg-body-light'"
          >
            <div class="block-content py-2" v-if="advancedBlockItems.length">
              <advanced-block-item
                v-for="item in advancedBlockItems"
                :key="item.label"
                :label="item.label"
                :value="item.value"
              />
            </div>
          </div>
        </b-col>
        <b-col cols="12">
          <bancor-checkbox
            v-model="notUsState"
            label="I am not a US citizen or domiciliary"
          />
        </b-col>
      </div>

      <action-modal-status
        v-if="txBusy || error || success"
        :error="error"
        :success="success"
        :step-description="
          sections.length ? sections[stepIndex].description : undefined
        "
      />

      <b-col cols="12">
        <main-button
          @click.native="initAction"
          :active="true"
          :label="confirmButton"
          :disabled="txBusy"
          :large="true"
        />
      </b-col>
    </b-row>
  </base-modal>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";
import { vxm } from "@/store/";
import {
  LiquidityModule,
  LiquidityParams,
  Step,
  ViewRelay,
  ViewReserve
} from "@/types/bancor";
import SelectPoolRow from "@/components/pool/SelectPoolRow.vue";
import BaseModal from "@/components/common/BaseModal.vue";
import PoolLogos from "@/components/common/PoolLogos.vue";
import AdvancedBlockItem from "@/components/common/AdvancedBlockItem.vue";
import MainButton from "@/components/common/Button.vue";

import { namespace } from "vuex-class";
import ActionModalStatus from "@/components/common/ActionModalStatus.vue";
import BancorCheckbox from "@/components/common/BancorCheckbox.vue";
import numeral from "numeral";

const bancor = namespace("bancor");

@Component({
  components: {
    BancorCheckbox,
    ActionModalStatus,
    AdvancedBlockItem,
    PoolLogos,
    BaseModal,
    SelectPoolRow,
    MainButton
  }
})
export default class ModalPoolAction extends Vue {
  @bancor.Action addLiquidity!: LiquidityModule["addLiquidity"];
  @bancor.Action removeLiquidity!: LiquidityModule["removeLiquidity"];

  @Prop() amountsArray!: string[];
  @Prop() selectedToken?: ViewReserve;
  @Prop() advancedBlockItems!: any[];
  numeral = numeral;

  txBusy = false;
  success = "";
  error = "";
  sections: Step[] = [];
  stepIndex = 0;

  notUsState: boolean = false;
  get slippageTolerance() {
    return vxm.bancor.slippageTolerance;
  }
  get confirmButton() {
    return this.error
      ? "Try Again"
      : this.success
      ? "Close"
      : this.txBusy
      ? "processing ..."
      : "Confirm";
  }

  get pool(): ViewRelay {
    return vxm.bancor.relay(this.$route.params.account);
  }

  get withdrawLiquidity(): boolean {
    return this.$route.params.poolAction === "remove";
  }

  get darkMode(): boolean {
    return vxm.general.darkMode;
  }

  setDefault() {
    this.sections = [];
    this.error = "";
    this.success = "";
    this.notUsState = false;
  }

  get isCountryBanned() {
    return vxm.general.isCountryBanned;
  }

  async initAction() {
    if (this.success) {
      this.$bvModal.hide("modal-pool-action");
      this.success = "";
      await this.$router.push({ name: "Pool" });
      return;
    }

    if (this.error) {
      this.error = "";
      return;
    }

    if (!this.notUsState || this.isCountryBanned) {
      this.error =
        "This action through swap.bancor.network is not available in your country.";
      return;
    }

    this.setDefault();

    const params: LiquidityParams = {
      id: this.pool.id,
      reserves: [],
      onUpdate: this.onUpdate
    };

    // Add AND Remove Liquidity V1
    if (!this.pool.v2) {
      params.reserves.push({
        id: this.pool.reserves[0].id,
        amount: this.amountsArray[1]
      });
      params.reserves.push({
        id: this.pool.reserves[1].id,
        amount: this.amountsArray[2]
      });
    } else {
      // Add V2
      if (this.$route.params.poolAction === "add" && this.selectedToken) {
        params.reserves.push({
          id: this.selectedToken.id,
          amount: this.amountsArray[1]
        });
        // Remove V2
      } else if (
        this.$route.params.poolAction === "remove" &&
        this.selectedToken
      ) {
        params.reserves.push({
          id: this.selectedToken.id,
          amount: this.amountsArray[0]
        });
      }
    }

    // Add Liquidity V2
    if (this.selectedToken && this.pool.v2) {
      // params.reserves.push({
      //   id: this.selectedToken.id,
      //   amount: this.amountsArray[1]
      // });
    } else {
      // Remove Liquidity V2
      if (this.selectedToken) {
        params.reserves.push({
          id: this.selectedToken.id,
          amount: this.amountsArray[0]
        });
      }
    }

    try {
      this.txBusy = true;
      let txResult: any;

      if (!this.withdrawLiquidity) txResult = await this.addLiquidity(params);
      else txResult = await this.removeLiquidity(params);
      this.success = txResult;
      // @ts-ignore
      this.$gtag.event(
        this.withdrawLiquidity ? "removeLiquidity" : "addLiquidity",
        {
          event_category: "success",
          event_label: this.$route.params.account
        }
      );
    } catch (e) {
      this.error = e.message;
      // @ts-ignore
      this.$gtag.event(
        this.withdrawLiquidity ? "removeLiquidity" : "addLiquidity",
        {
          event_category: "error",
          event_label: this.$route.params.account
        }
      );
    } finally {
      this.txBusy = false;
    }
  }

  onUpdate(stepIndex: number, steps: Step[]) {
    this.stepIndex = stepIndex;
    this.sections = steps;
  }
}
</script>
<style lang="scss">
.modal-body {
  padding-top: 0 !important;
}
</style>
