<template>
  <div class="illust-wrap">
    <div class="illusts">
      <TopBar />
      <FavoriteIllusts :id="+$route.params.id" key="multi-favorite" />
    </div>
  </div>
</template>
<script>
import TopBar from "@/components/TopBar";
import FavoriteIllusts from "./components/FavoriteIllusts";
export default {
  name: 'FavoriteIllustsFull',
  components: {
    TopBar,
    FavoriteIllusts
  },
}
</script>
<style lang="stylus" scoped>
.illust-wrap
  height: 100vh

  ::v-deep .top-bar-wrap
    width 30%
    padding-top 20px
    background transparent

.illusts
  margin-top 10px
  padding-bottom 60px

</style>
