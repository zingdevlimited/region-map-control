<template>
  <div class="home">
    <input type="text" placeholder="token" v-model="token" />
    <div class="text-areas">
      <textarea cols="80" rows="50" v-model="queryText"></textarea>
      <textarea cols="80" rows="50" readonly v-model="result"></textarea>
    </div>
    <button @click.prevent="runQuery">GO</button>
  </div>
</template>

<style lang="scss" scoped>
.home {
  display: flex;
  flex-direction: column;

  .text-areas {
    display: flex;

    textarea {
      flex-grow: 1;
    }
  }
}
</style>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

@Component({})
export default class Home extends Vue {
  private token = '';
  private queryText = '';
  private result = '';

  protected created() {
    const lastToken = localStorage.getItem('last-token');
    if (lastToken) { this.token = lastToken; }
    const lastQuery = localStorage.getItem('last-query');
    if (lastQuery) { this.queryText = lastQuery; }
  }

  protected async runQuery() {
    if (this.token !== '' && this.queryText !== '') {
      localStorage.setItem('last-token', this.token);
      localStorage.setItem('last-query', this.queryText);
      const cleanQuery = this.queryText.replace(/[\r\n]/g, '');
      const result = await fetch(
        `https://api-v1.prospect365-dev.com/${cleanQuery}`,
        {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        }
      );

      this.result = JSON.stringify((await result.json()).value, null, 2);
    }
  }
}
</script>
