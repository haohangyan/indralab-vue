<template>
  <div class="col-12 col-lg-10">
    <label :for="inputId" class="d-flex align-items-center mb-2">
      {{ labelText }}
      <span
        :class="isOther ? 'orange-dot role-dot' : 'blue-dot role-dot'"
        aria-hidden="true"
      ></span>
      <template v-if="isOther">
        <button
          type="button"
          class="btn btn-sm btn-light btn-link btn-with-tooltip ml-1"
          data-toggle="collapse"
          @click="isCollapsed = !isCollapsed"
          :data-target="'#collapse-' + inputId"
        >
          <i class="fas fa-chevron-down fa-sm" :class="{ 'fa-rotate-180': !isCollapsed }"></i>
            Add other agent
          <span class="info-icon">?</span>
          <div class="tooltip-box text-wrap">If you want to use two agents in the search, add the second agent here.</div>
        </button>
      </template>
    </label>

    <template v-if="!options || options_empty">
      <div class="collapse-container" :class="{ 'collapse': isOther, 'show': !isOther }" :id="'collapse-' + inputId">
        <div class="d-flex align-items-center flex-wrap flex-md-nowrap agent-row">
          <input
            class="form-control flex-grow-1"
            :id="inputId"
            type="text"
            v-model="agent_str"
            :placeholder="isOther
              ? `Enter name or identifier (e.g. 'MEK', 'FPLX:MEK', 'hgnc:3321' or 'selumetinib')`
              : `Enter name or identifier (e.g. 'MEK', 'FPLX:MEK', 'hgnc:3321' or 'selumetinib')`"
          />
          <button
            type="button"
            class="btn btn-secondary ml-lg-2 mt-2 mt-lg-0 btn-with-tooltip agent-action-button"
            @click="lookupOptions"
          >
            Find identifier
            <span class="info-icon">?</span>
            <div class="tooltip-box text-wrap">
              Use this button to ground the entered name with
              <a href="https://grounding.indra.bio/" target="_blank">Gilda</a>.
            </div>
          </button>
        </div>
      </div>
      <div class="mt-1">
        <small v-if="agent_str && hasSearched" class="form-text text-muted">{{ hintText }}</small>
        <small v-if="hasSearched && searching" class="text-muted d-block">Searching...</small>
        <small v-if="hasSearched && options_empty" class="text-warning d-block">No groundings found...</small>
        <small
          v-if="hasSearched && search_error"
          class="text-danger d-block"
          :title="search_error"
          style="cursor: default"
        >Search failed!</small>
      </div>
    </template>

    <div v-else-if="options.length === 1" class="mt-2">
      <small class="text-muted d-block mb-1">GILDA grounding</small>
      <div class="d-flex align-items-center flex-wrap flex-md-nowrap agent-row">
        <div class="form-control gilda-dropdown flex-grow-1" v-html="printOption(options[0])"></div>
        <button type="button" class="btn btn-outline-secondary ml-lg-2 mt-2 mt-lg-0 agent-action-button" @click="resetOptions">
          Change
        </button>
      </div>
    </div>

    <div v-else class="mt-2">
      <small class="text-muted d-block mb-1">GILDA grounding</small>
      <div class="d-flex align-items-center flex-wrap flex-md-nowrap agent-row">
        <select class="custom-select gilda-dropdown flex-grow-1" v-model="selected_option_idx">
          <option :value="-1" selected disabled hidden>Select grounding option...</option>
          <option
            v-for="(option, option_idx) in options"
            :key="option_idx"
            :value="option_idx"
            v-html="printOption(option)"
          ></option>
        </select>
        <button type="button" class="btn btn-outline-secondary ml-lg-2 mt-2 mt-lg-0 agent-action-button" @click="resetOptions">
          Cancel
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AgentSelect',
  props: [
    'value',
    'exampleTick',
    'isOther',
    'labelText',
    'inputId'
  ],
  data () {
    return {
      agent_str: '',
      searching: false,
      options: null,
      selected_option_idx: -1,
      search_error: null,
      hasSearched: false,
      isCollapsed: true,
    }
  },
  methods: {
    async lookupOptions () {
      if (!this.agent_str) {
        alert('Please enter an agent string...')
        this.searching = false
        this.hasSearched = false
        return
      }
      if (!this.$ground_url) {
        this.search_error = 'Grounding service URL is not configured.'
        this.hasSearched = true
        return
      }
      this.search_error = null
      this.hasSearched = true
      this.searching = true
      try {
        const url = `${this.$ground_url}?agent=${encodeURIComponent(this.agent_str)}`
        const resp = await fetch(url, { method: 'GET' })
        const contentType = resp.headers.get('content-type') || ''

        if (!resp.ok) {
          this.search_error = `(${resp.status}) ${resp.statusText}`
          this.options = []
        } else if (!contentType.includes('application/json')) {
          this.search_error = 'Unexpected response from grounding service.'
          this.options = []
        } else {
          this.options = await resp.json()
          if (this.options.length === 1) this.selected_option_idx = 0
          this.search_error = null
        }
      } catch (err) {
        this.search_error = err && err.message ? err.message : 'Grounding lookup failed.'
        this.options = []
      } finally {
        this.searching = false
      }
    },
    resetOptions () {
      this.options = null
      this.selected_option_idx = -1
      this.hasSearched = false
      this.search_error = null
      this.searching = false
    },
    printOption (option) {
      const term = option.term
      return `<b>${term.entry_name}</b> (score: ${option.score.toFixed(2)}, ${term.id} from ${term.db})`
    }
  },
  computed: {
    displayText () {
     if (this.options && !this.options_empty && this.selected_option_idx >= 0) {
       return this.options[this.selected_option_idx].term.entry_name;
     }
     return (this.agent_str || '').trim();
   },
    options_empty () {
      if (!this.options) return false
      return this.options.length === 0
    },

    parsedPrefix () {
      const text = (this.agent_str || '').trim()
      const m = text.match(/^(hgnc|fplx)\s*[：: ]?\s*(.+)?$/i)
      if (!m) return null
      const ns = m[1].toUpperCase()
      const id = (m[2] || '').trim() || null
      return { ns, id }
    },

    detectedNamespace () {
      return this.parsedPrefix ? this.parsedPrefix.ns : null
    },

    agent_id_from_input () {
      if (this.parsedPrefix) return this.parsedPrefix.id
      return (this.agent_str || '').trim()
    },

    namespace_from_input () {
      if (this.parsedPrefix) return this.parsedPrefix.ns
      return 'AUTO'
    },

    partialConstraint () {
      let ret = null;

      if (this.options && !this.options_empty && this.selected_option_idx >= 0) {
        const chosen = this.options[this.selected_option_idx];
        ret = { agent_id: chosen.term.id, namespace: chosen.term.db };
      } else if (this.agent_id_from_input) {
        ret = { agent_id: this.agent_id_from_input, namespace: this.namespace_from_input };
      }

      return ret;
    },

    hintText () {
      const p = this.parsedPrefix
      if (p && p.ns === 'HGNC') {
        return p.id ? `Searching by HGNC id ${p.id}` : 'Searching by HGNC id'
      }
      if (p && p.ns === 'FPLX') {
        return p.id ? `Searching by FPLX id ${p.id}` : 'Searching by FPLX id'
      }
      return 'Searching by Text name'
    },

  },
  watch: {
    exampleTick () {
      const v = this.value || {};
      const ns = typeof v.namespace === 'string' ? v.namespace.toUpperCase() : null;
      const id = typeof v.agent_id === 'string' ? v.agent_id : '';
      if (ns && id && ns !== 'AUTO') {
        this.agent_str = `${ns.toLowerCase()}:${id}`;
      } else {
        this.agent_str = id;
      }
      this.options = null;
      this.selected_option_idx = -1;
      this.search_error = null;
      this.hasSearched = false;
    },
    agent_str (val, oldVal) {
      if (val === oldVal) return;
      this.hasSearched = false;
      this.search_error = null;
      this.options = null;
      this.selected_option_idx = -1;
      this.searching = false;
    },
      partialConstraint (pc) {
      if (!pc) return;
      const base = (this.value && typeof this.value === 'object') ? this.value : {};
      const merged = { ...base, ...pc };
      this.$emit('input', merged);
      this.$emit('display', this.displayText);
    },
  }
}
</script>
<style scoped>
  .gilda-dropdown {
    width: 100%;
    min-width: 0;
  }

  .collapse-container {
    overflow: visible;
  }

  .collapse-container:not(.show) {
    display: none !important;
  }

  .agent-row {
    gap: 8px;
  }

  .agent-action-button {
    min-width: 170px;
    flex-shrink: 0;
  }

  .btn-with-tooltip {
    position: relative;
    padding-right: 34px;
    overflow: visible;
    white-space: nowrap;
  }

  .btn-with-tooltip,
  .btn-with-tooltip .info-icon {
    cursor: pointer;
  }

  .btn-with-tooltip .info-icon {
    position: absolute;
    right: 8px;
    top: 50%;
    transform: translateY(-50%);
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #007bff;
    color: #fff;
    font: 700 12px/18px sans-serif;
    text-align: center;
    border: 1px solid rgba(0, 0, 0, 0.15);
  }

  .btn-with-tooltip .info-icon::after {
    content: "";
    position: absolute;
    left: 100%;
    top: -8px;
    bottom: -8px;
    width: 32px;
  }

  .btn-with-tooltip .tooltip-box {
    position: absolute;
    left: calc(100% + 8px);
    top: 50%;
    transform: translateY(-50%);
    min-width: 240px;
    background: #f8f9fa;
    color: #333;
    border: 1px solid #ddd;
    border-radius: 6px;
    padding: 8px 10px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
    z-index: 1000;
    visibility: hidden;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.12s;
  }

  .btn-with-tooltip .info-icon:hover + .tooltip-box,
  .btn-with-tooltip .tooltip-box:hover {
    visibility: visible;
    opacity: 1;
    pointer-events: auto;
  }

  .blue-dot {
    background-color: #6fa8dc;
  }

  .orange-dot {
    background-color: #ff9900;
  }

  .role-dot {
    width: 14px;
    height: 14px;
    border-radius: 50%;
    display: inline-block;
    margin-right: 3px;
    margin-left: 6px;
    vertical-align: middle;
    margin-bottom: 2px;
  }
</style>