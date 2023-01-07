<template>
  <div class="container">
    <h2 class="title-airdrop">Airdrop</h2>
    <h1 class="sub-title-airdrop">My progress</h1>

    <div class="flex flex-wrap p-4">
      <SpProgressbar
        size="large"
        :val="missionProgress.value"
        :text="`${missionProgress.value}%`"
        text-align="right"
      >
      </SpProgressbar>
    </div>

    <h2 class="title-airdrop">My Missions</h2>

    <div
      v-for="(mission, index) in missionSteps"
      class="mission-group text-stone-100"
    >
      <div class="col-md-6">
        <p class="mission-title">Mission #{{ index + 1 }}</p>
        <p class="mission-content">
          <a :href="mission.relevant_url" target="_blank">{{
            mission.description
          }}</a>
        </p>
      </div>
      <div>
        <h4 v-if="missionStepText(index) == 0" class="text-mission-completed">
          Complete
        </h4>
        <h4 v-if="missionStepText(index) == 1" class="text-mission-uncompleted">
          Not Complete
        </h4>
        <Button
          v-if="missionStepText(index) == 2"
          class="Button"
          @click="() => doClaim(index)"
        >
          Claim
        </Button>
        <SpSpinner v-if="missionStepText(index) == -1" />
      </div>
    </div>

    <p class="airdrop-notif">
      You can claim your tokens only when the previous step is completed
    </p>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, reactive, watch } from "vue";
import { useStore } from "vuex";
import { SpProgressbar } from "../../components_token";
import { useAddress, useAssets } from "../../composables";
import { Amount } from "../../utils/interfaces";
import { useLoading } from "vue-loading-overlay";
import { useToast } from "vue-toastification";

import "vue-loading-overlay/dist/css/index.css";
import SpSpinner from "../../components_token/SpSpinner/SpSpinner.vue";

const $s = useStore();
const $loading = useLoading({});
const toast = useToast();

const missionSteps = reactive([
  {
    description: "Initial claim",
    relevant_url: "#",
    claimable: false,
    completed: false,
  },
  {
    description: "Stake your TMUN (delegate to a validator)",
    relevant_url: "https://staking.mun.money",
    claimable: false,
    completed: false,
  },
  {
    description: "Vote for a governance proposal",
    relevant_url: "#",
    claimable: false,
    completed: false,
  },
  {
    description: "Make a swap on AMM(TMUN to DGM)",
    relevant_url: "https://swap.mun.money",
    claimable: false,
    completed: false,
  },
]);

const missionProgress = reactive({ value: 0 });
const claimStatus = reactive({
  loading: false,
});

let { address } = useAddress({ $s });
// let { balances } = useAssets({ $s });

const missionStepText = (index: number): number => {
  if (claimStatus.loading == true) return -1;
  if (missionSteps[index].completed == true) return 0;
  else {
    if (missionSteps[index].claimable == false) return 1;
    if (missionSteps[index].claimable == true) return 2;
  }
  return -1;
};

type ClaimRecordType = {
  claim_record: {
    action_completed?: boolean[];
    action_ready?: boolean[];
    address?: string;
    initial_claimable_amount?: Amount[];
  };
};

type ClaimableForActionResponse = {
  coins: Amount[];
};

const initMissionStatus = async () => {
  for (let i = 0; i < 4; i++) {
    missionSteps[i].claimable = missionSteps[i].completed = false;
  }
};

const fetchMissionStatus = async () => {
  initMissionStatus();
  if (address.value == "") return;
  claimStatus.loading = true;

  const cr: ClaimRecordType = await $s.dispatch(
    "mun.claim.v1beta1/QueryClaimRecord",
    {
      params: { address: address.value },
      options: { subscribe: true },
    }
  );

  const claimableForInitialAction: ClaimableForActionResponse =
    await $s.dispatch("mun.claim.v1beta1/QueryClaimableForAction", {
      params: {
        address: address.value,
        action: "ActionInitialClaim",
      },
    });

  const { action_completed, action_ready } = cr.claim_record;
  for (let i = 0; i < 4; i++) {
    missionSteps[i].completed =
      action_completed && action_completed.length > i
        ? action_completed[i]
        : false;
    missionSteps[i].claimable =
      action_ready && action_ready.length > i ? action_ready[i] : false;
  }
  if (claimableForInitialAction.coins.length > 0) {
    missionSteps[0].claimable = true;
  }
  const prog = action_completed?.findIndex((completed) => completed == false)!;
  missionProgress.value = prog == -1 ? 0 : 25 * prog;

  claimStatus.loading = false;
};

onMounted(() => {
  fetchMissionStatus();
});

watch(address, () => {
  fetchMissionStatus();
});

const doClaim = async (missionId: number) => {
  const loader = $loading.show({
    color: "#BBB",
    backgroundColor: "black",
  });

  try {
    let payload = {
      sender: address.value,
      address: address.value,
      action: missionId,
    };

    const txResult = await $s.dispatch("mun.claim.v1beta1/sendMsgClaimFor", {
      value: payload,
    });
    console.log(txResult);

    if (txResult.code != 0) {
      throw new Error();
    }

    toast.success("Success");
    loader.hide();
    fetchMissionStatus();
  } catch (e) {
    loader.hide();
    toast.error("Failed");
    console.error(e);
  }
};
</script>

<style lang="scss" scoped>
$base-color: rgba(0, 0, 0, 0.03);
$animation-duration: 1.6s;
$shine-color: rgba(0, 0, 0, 0.06);
$avatar-offset: 32 + 16;

.text-mission-completed {
  font-size: 18px;
  margin: 0;
  color: rgb(87, 206, 122);
}

.text-mission-uncompleted {
  font-size: 18px;
  margin: 0;
  color: rgb(192, 71, 112);
}

.assets-header {
  display: flex;
  flex-wrap: wrap;
  width: 100%;
}

.title-airdrop {
  font-family: Inter, serif;
  font-style: normal;
  font-weight: 700;
  font-size: 28px;
  line-height: 127%;
  /* identical to box height, or 36px */

  letter-spacing: -0.02em;
  font-feature-settings: "zero";

  color: var(--background-color-lila);
  margin-top: 2%;
}

.sub-title-airdrop {
  font-family: Inter, serif;
  font-style: normal;
  font-size: 18px;
  margin-top: 2%;
  line-height: 127%;
  /* identical to box height, or 36px */
  padding: 10px;
  font-feature-settings: "zero";
  color: var(--background-color-lila);
}

.mission-title {
  font-family: Inter, serif;
  font-style: normal;
  font-weight: 700;
  font-size: 14px;
  /* identical to box height, or 36px */
  margin-top: 15px;
  font-feature-settings: "zero";
  color: var(--background-color-lila);
  margin-top: 0;
}

.mission-content {
  font-family: Inter, serif;
  font-style: normal;
  font-weight: 700;
  font-size: 18px;
  // line-height: 127%;
  margin: 0;
  /* identical to box height, or 36px */

  // font-feature-settings: 'zero';

  a {
    text-decoration: none;
    color: var(--background-color-lila);

    &:hover {
      text-decoration: underline;
    }
  }
}

.mission-group {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 2%;
  border: rgba(151, 130, 130, 0.1);
  border-radius: 10px;
  border: 1px solid grey;
  padding: 22px 20px;
  // padding-left: 10px;
}

.claim-button-style {
  display: flex;
  justify-content: space-between;
  height: auto;
  width: auto;
  font-size: 14px;
  background: var(--background-color-lila);
  border-radius: 5px;
  padding: 5px;
}

.Button {
  background: var(--background-color-lila);
  width: 80px;
  border: none;
  color: #fff;
  cursor: pointer;
  font-size: 14px;
  // margin-top: 5px;
  // margin-right: 20px;
  padding: 12px 20px;
  border-radius: 15px;
  border-color: var(--background-color-lila);
  transition: background 0.2s ease-in-out;

  &:hover {
    background: darken(#ac61d7, 20);
  }

  &:disabled {
    opacity: 0.5;
  }
}

.airdrop-notif {
  font-size: 14px;
  letter-spacing: 0.05rem;
  color: #888;
  margin-top: 15px;
  display: none;
}

.feedback {
  position: absolute;
  top: 400px;
  left: 48%;
  z-index: 9999999999;
  background: wheat;
  padding: 10px;
}
</style>
