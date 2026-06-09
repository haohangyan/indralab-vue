<template>
  <span class="agent-pair">
    <div class="row header"
         :style="`cursor: ${(searching) ? 'progress': 'pointer'};`"
         @click="toggleRelations">
      <div class='col text-left'>
        <h5>
          <span v-html="english"></span>
          <small v-if='cur_count'
                 class='badge badge-success badge-pill'>
            &#9998; {{ cur_count }}
          </small>
          <span v-if="affect_summary"
                class="consensus-affect-summary">
            <span v-for="(step, index) in affect_summary.steps"
                  :key="`${step}-${index}`"
                  class="consensus-affect-step">
              <span v-if="index > 0"
                    class="consensus-affect-arrow">
                &rarr;
              </span>
              <span class="consensus-affect-chip">
                {{ step }}
              </span>
            </span>
          </span>
        </h5>
      </div>
      <div class="col-auto text-right">
        <source-display :source_counts="source_counts"></source-display>
      </div>
    </div>
    <div class="error-message" v-show="search_failed">
      <i style="color: red">Failed to find statements: {{search_failed}}</i>
    </div>
    <div class="row stmt_list" v-show="show_relations">
      <div class="col">
        <div class="container right-bar">
          <span v-for="relation in list_shown" :key="relation.id">
            <relation v-bind="relation" :context_queries="context_queries"></relation>
          </span>
          <div class='text-center clickable'
               :style="`cursor: ${(searching) ? 'progress' : 'pointer'}`"
               v-show='show_buttons'
               @click='loadMore'>
            Load {{ next_batch }} more...
          </div>
          <div class='text-center clickable'
               :style="`cursor: ${(searching) ? 'progress' : 'pointer'}`"
               v-show='show_buttons'
               @click='loadAll'>
            Load all {{ base_list ? base_list.length : 0 - end_n }} remaining...
          </div>
        </div>
      </div>
    </div>
  </span>
</template>

<script>
  import piecemeal_mixin from '../piecemeal_mixin'

  export default {
    name: "AgentPair",
    props: {
      english: String,
      source_counts: Object,
      agents: Object,
      affect_summary: Object,
      cur_count: {
        type: Number,
        default: 0
      },
      context_queries: Array,
      hashes: Array,
    },
    data: function() {
      return {
        show_relations: false,
        relations: null,
        searching: false,
        evidence_counts: null,
        search_failed: null,
      }
    },
    methods: {
      toggleRelations: function() {
        if (this.relations == null)
          this.getRelations();
        this.show_relations = !this.show_relations;
      },

      getRelations: async function() {
        if (this.searching)
          return false;

        window.console.log("Getting relations.");
        this.searching = true;

        let query_data = {
          agent_json: this.agents,
          hashes: this.hashes,
        };

        let query_strs = [];
        query_strs.push("with_cur_counts=true");

        let query_str = query_strs.join('&');
        window.console.log(this.$relation_url);
        window.console.log(query_str);
        window.console.log(query_data);
        const resp = await fetch(
          this.$relation_url + '?' + query_str,
          {
            method: 'POST',
            body: JSON.stringify(query_data),
            headers: {'Content-Type': 'application/json'}
        });
        let success = true;
        if (resp.status === 200) {
          const resp_json = await resp.json();
          window.console.log(resp_json);

          this.relations = resp_json.relations;
          this.evidence_counts = resp_json.evidence_counts;
          this.search_failed = null;
        } else {
          this.search_failed = `(${resp.status}) ${resp.statusText}`
          success = false;
        }
        this.searching = false;
        return success;
      }
    },
    computed: {
      base_list: function() {
        return this.relations;
      }
    },
    mixins: [piecemeal_mixin]
  }
</script>

<style scoped>
  .header {
    cursor: pointer;
  }
  .header:hover {
    background-color: #e0e0e9;
  }
  .right-bar {
    border-right: #0d5aa7 solid 1px;
  }
  .consensus-affect-summary {
    display: inline-flex;
    align-items: center;
    gap: 0.15rem;
    margin-left: 0.75rem;
    color: #555;
    font-size: 0.58em;
    font-weight: 500;
    white-space: nowrap;
    vertical-align: middle;
  }
  .consensus-affect-chip {
    display: inline-block;
    padding: 0.1rem 0.35rem;
    border: 1px solid #ddd;
    border-radius: 3px;
    background-color: #fafafa;
  }
  .consensus-affect-step {
    display: inline-flex;
    align-items: center;
    gap: 0.15rem;
  }
  .consensus-affect-arrow {
    display: inline-block;
    margin: 0 0.1rem;
    color: #777;
    font-weight: 600;
  }
</style>
