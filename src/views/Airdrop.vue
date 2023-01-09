<template>
  <div class="container">
    <div class="row row-sm-revers">
      <div>
        <SpAirdrop />
      </div>
    </div>
  </div>
</template>
<script>
import { SpAirdrop } from "../components";
import { computed } from "vue";
import { useStore } from "vuex";

export default {
  name: "Airdrop",

  components: { SpAirdrop },

  setup() {
    // store
    let $s = useStore();

    // computed
    let address = computed(() => $s.getters["common/wallet/address"]);

    return {
      address,
    };
  },
};
</script>

<style scoped></style>
