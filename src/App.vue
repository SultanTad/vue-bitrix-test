<template>
<!--   
    <v-overlay :value="isPreloader">
      <span>Идет загрузка...</span>
    </v-overlay>
     -->
    <div>
      <router-view/>
    </div>

  
</template>

<script>


export default {
  name: 'App',

  components: {},
  async created() {

    this.$eventBus.$on("preloader", () => this.isPreloader = !this.isPreloader);



  },
  methods: {

  },
  computed: {
    window: () => window,
  },
  data: () => ({
    isPreloader: false,
  }),
};
</script>
<style>
body, html {
  overflow-y: auto !important;
}

.v-data-table > .v-data-table__wrapper tbody tr.v-data-table__expanded__content {
  box-shadow: none !important;
}

</style>
