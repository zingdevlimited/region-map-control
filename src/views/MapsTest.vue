<template>
  <div class="holder">
    <region-map v-model="exampleData" />
    <ul>
      <li v-for="area in exampleData" :key="area.AreaCode">
        {{ area.AreaCode }}
        <ul>
          <li v-for="outcode in area.OutCodes" :key="outcode">{{ outcode }}</li>
        </ul>
      </li>
    </ul>
  </div>
</template>

<style lang="scss" scoped>
.holder {
  display: flex;
}
</style>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator';
import { Area } from '@/components/RegionMap.vue';

@Component({
  components: {
    'region-map': () =>
      import(/* webpackChunkName: "RegionMap" */ '@/components/RegionMap.vue'),
  },
})
export default class MapsTest extends Vue {
  private exampleData: Area[] = [];

  protected created() {
    this.exampleData.push({
      AreaCode: 'PE',
      OutCodes: ['PE29', 'PE14', 'PE15'],
    });
  }
}
</script>
