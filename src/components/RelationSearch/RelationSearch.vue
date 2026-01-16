<template>
  <div class='relation-search nvm'
       :style="`cursor: ${(searching) ? 'progress': 'auto'};`">
    <div class="search-form-section">
      <div id="search-row">
        <div class="nav-btn">
          <h4>
            Statement search
            <button class="btn"
                    :disabled="cannotGoBack"
                    @click="backButton">
              &lt; Back
            </button>
            <button class="btn"
                    :disabled="cannotGoForward"
                    @click="forwardButton">
              Forward &gt;
            </button>
          </h4>
        </div>
      <div id="seach-box">
      <div class="container pl-0">
        <form>
          <div class="form-row"
              v-for="(pair, i) in hasAgentConstraints"
              :key="pair.idx">

            <agent-select
              :key="'agent-'+pair.idx"
              :inputId="i === 0 ? 'agent1' : 'agent2'"
              v-model="pair.c.constraint"
              :exampleTick="exampleTick"
              :labelText="i === 0 ? 'Agent (the name or identifier of a gene, family/complex, small molecule, biological process, etc.)' : 'Other agent (optional)'"
              :isOther="i >= 1"
              @display="setDisplay(pair.idx, $event)"
            ></agent-select>

            <button
              class="btn btn-sm btn-outline-danger"
              v-if="i >= 2"
              @click="removeConstraint(pair.idx)"
              style="margin-left:6px"
              title="Remove this agent">
              ×
            </button>
          </div>
        </form>
      </div>
      <div class="container pl-0">
        <form>
          <div class="form-row">
            <span>Agent role</span>
            <div class="role-presets">
              <button type="button"
                      class="btn-role"
                      :class="{ active: currentPreset === 'any-any' }"
                      @click.prevent="presetRoles('any-any')">
                <img :src="roleBtn3" />
                <span class="role-text">Subject/Object</span>
              </button>

              <button type="button"
                      class="btn-role"
                      :class="{ active: currentPreset === 's-o' }"
                      @click.prevent="presetRoles('s-o')">
                <img :src="roleBtn2" />
                <span class="role-text">Subject</span>
              </button>

              <button type="button"
                      class="btn-role"
                      :class="{ active: currentPreset === 'o-s' }"
                      @click.prevent="presetRoles('o-s')">
                <img :src="roleBtn1" />
                <span class="role-text">Object</span>
              </button>
            </div>
          </div>
        </form>
      </div>

      <div
         v-for="pair in nonAgentConstraints"
         :key="pair.idx">

        <!-- blank slot: choose what to add (your current code) -->
        <template v-if="!pair.c.class">
        <span>More filters:</span>
        <div class="form-inline">
          <select class="form-control"
                  name="constraint-select"
                  @input="reactToConstraintSelection($event, pair.idx)">
            <option :value="null" selected hidden>select filters...</option>
            <option v-for="(type_val, type_name) in constraint_classes"
                    :key="type_name"
                    :value="type_val">
              Filter by {{ type_name === 'mesh' ? (type_name + ' ID') : type_name }}
            </option>
          </select>
        </div>
      </template>

      <!-- filled constraint (non-agent only) -->
      <template v-else>
        <template v-if="pair.c.class === 'HasType'">
          <div class="container pl-0">
            <form>
              <div class="form-row">
                <span>Relation type:</span>
                <type-select v-model="pair.c.constraint"></type-select>
              </div>
            </form>
          </div>
        </template>

        <template v-else-if="pair.c.class === 'FromMeshIds'">
          <div class="container pl-0">
            <form>
              <div class="form-row">
                <mesh-select v-model="pair.c.constraint" :example-tick="exampleTick"></mesh-select>
              </div>
            </form>
          </div>
        </template>

        <template v-else-if="pair.c.class === 'FromPapers'">
          <div class="form-inline">
            <b>Paper:</b>
          <paper-select v-model="pair.c.constraint"></paper-select>
          <button
            class="btn btn-sm btn-outline-danger"
            @click="removeConstraint(pair.idx)"
            style="margin-left:6px"
            title="Remove Paper constraint"
          >×</button>
          </div>
        </template>

        <template v-else>
          <b style="color: red;">Developer error: unhandled constraint.class.</b>
        </template>
      </template>
        </div>


        <div>
          <button class="btn btn-primary"
                @click="searchButton"
                :disabled="searching">
          Search
        </button>
         <button v-if="lastSearchOk"
                 class="btn btn-outline-secondary"
                 style="margin-left:6px"
                 @click="copyLink">
           Copy Link
         </button>
        </div>
      </div>
      </div>

      <div id="error-box" class="nvm" v-show="search_error">
        <hr>
        <i style="color: red">Failed to load search results: {{ search_error }}.</i>
      </div>
    </div>
    <div class="results-section">
      <div id='result-box' class='nvm' v-if='!empty_relations'>
        <hr>
        <h4>Results</h4>
        <small>I found statements that {{ query_string }}</small>
        <hr>
        <h4 v-show='empty_relations & search_history'>Nothing found.</h4>
        <div id="result-list">
          <span v-for="agent_pair in list_shown" :key="agent_pair.id">
            <agent-pair v-bind="agent_pair" :context_queries="context_queries"></agent-pair>
          </span>
        </div>
        <span v-show="searching">Loading...</span>
      </div>
      <div v-else-if="agent_pairs !== null">
        No results found.
      </div>
    </div>
  </div>
</template>

<script>
  import piecemeal_mixin from "../piecemeal_mixin";
  import roleBtn1 from '@/image/role_button1.png'
  import roleBtn2 from '@/image/role_button2.png'
  import roleBtn3 from '@/image/role_button3.png'
  export default {
    name: "RelationSearch",
    props: {
      autoload: {
        type: Boolean,
        default: true
      }
    },
    data: function() {
      return {
        new_const_type: null,
        constraints: {},
        cidx: 0,
        constraint_classes: {
          paper: 'FromPapers'
        },
        context_queries: null,
        agent_pairs: null,
        show_search: true,
        searching: false,
        query_string: null,
        next_offset: 0,
        search_error: null,
        search_history: [],
        history_idx: -1,
        complexes_covered: null,

        exampleTick: 0,
        roleBtn1,
        roleBtn2,
        roleBtn3,
        shareUrl: "",
        lastSearchOk: false,
        displayTextMap: {},
      }
    },
    methods: {
      addConstraint (constraint_class) {
        // used when you pre-add Agent/Type at created()
        let def = null;
        if (constraint_class === 'HasAgent')    def = {};
        else if (constraint_class === 'HasType') def = { stmt_types: [] };
        else if (constraint_class === 'FromMeshIds') def = { mesh_ids: [] };
        else if (constraint_class === 'FromPapers')  def = { paper_list: [] };

        this.$set(this.constraints, this.cidx, {
          class: constraint_class,
          constraint: def,
          inverted: false
        });
        this.cidx++;
      },
      addBlankSlot () {
        this.$set(this.constraints, this.cidx, {
          class: null,            // ← shows the "select constraint..." row
          constraint: null,
          inverted: false
        });
        this.cidx++;
      },
      ensureBlankSlot () {
        const hasBlank = Object.values(this.constraints).some(c => c && !c.class);
        if (!hasBlank) this.addBlankSlot();
      },

      reactToConstraintSelection (event, idx) {
        const chosen = event.target.value; // 'FromMeshIds' or 'FromPapers'
        let def = null;
        if (chosen === 'FromMeshIds') def = { mesh_ids: [] };
        if (chosen === 'FromPapers')  def = { paper_list: [] };

        this.$set(this.constraints[idx], 'class', chosen);
        this.$set(this.constraints[idx], 'constraint', def);
        this.ensureBlankSlot();
      },
      removeConstraint: function(constraint_idx) {
        this.$delete(this.constraints, constraint_idx)
        this.ensureBlankSlot();
      },

      async searchButton() {
        const active = Object.values(this.constraints).filter(c => this.isFilled(c));
        if (active.length === 0) {
          alert('Please enter at least one constraint.');
          return;
        }

        // Validate MeSH IDs if MeshSelect has input
        const meshValidationError = this.validateMeshIds();
        if (meshValidationError) {
          alert(meshValidationError);
          return;
        }

        this.lastSearchOk = false;
        this.shareUrl = "";
        this.next_offset = 0;
        this.agent_pairs = null;
        this.complexes_covered = null;
        this.pushHistory?.(); // harmless if you later remove history
        return await this.search();
      },

      search: async function() {
        // Don't run multiple searches at once.
        if (this.searching)
          return false;

        // If the database says there is nothing more to find, stop looking.
        if (this.next_offset == null) {
          window.console.log("Offset is null, ignoring search.");
          return false;
        }

        this.searching = true;
        let query_strs = [];
        this.context_queries = [];

        // Format the constraints into the query.
        let query_jsons = [];
        let cumulative_queries = {};
        for (let idx in this.constraints) {
          const param = this.constraints[idx];
          if (!this.isFilled(param)) continue; // ← skip empty constraints

          if (param.class === 'HasAgent') {
            query_jsons.push(param);
          } else {
            for (let [class_name, list_name] of [
              ['HasType', 'stmt_types'],
              ['FromMeshIds', 'mesh_ids'],
              ['FromPapers', 'paper_list']
            ]) {
              if (class_name !== param.class) continue;
              if (cumulative_queries[class_name]) {
                // merge lists
                cumulative_queries[class_name].constraint[list_name] =
                  cumulative_queries[class_name].constraint[list_name]
                    .concat(param.constraint[list_name]);
              } else {
                cumulative_queries[class_name] = param;
                query_jsons.push(param);
              }
            }
          }
        }

        // Build up the context queries.
        this.context_queries = [];
        for (let cn in cumulative_queries) {
          let constraint = cumulative_queries[cn].constraint;
          if (cn === 'FromPapers') {
            let paper_strings = [];
            for (let [id_type, paper_id] of constraint.paper_list){
              paper_strings.push(`${paper_id}@${id_type}`)
            }
            this.context_queries.push(`paper_ids=${paper_strings.join(',')}`);
          } else if (cn === 'FromMeshIds') {
            this.context_queries.push(`mesh_ids=${constraint.mesh_ids.join(',')}`)
          }
        }

        let query;
        if (query_jsons.length === 1)
          query = query_jsons[0];
        else
          query = {
            class: 'Intersection',
            constraint: {query_list: query_jsons},
            inverted: false
          }

        query_strs.push('limit=50');
        query_strs.push(`offset=${this.next_offset}`);
        query_strs.push('with_cur_counts=true');
        query_strs.push('with_english=true');
        query_strs.push('with_hashes=true');
        query_strs.push('format=json-js');
        let query_body = JSON.stringify({
          query: query,
          complexes_covered: this.complexes_covered
        });
        window.console.log(`Query params: ${query_strs}`);
        window.console.log(`JSON query: ${query_body}`);
        window.console.log(`Context queries: ${this.context_queries}`);

        // Make the query
        let url = this.$agent_url + '?' + query_strs.join('&');
        window.console.log(url);
        const resp = await fetch(url, {
          method: 'POST',
          body: query_body,
          headers: {'Content-Type': 'application/json'}
        });

        // Check that the query is good (exit if not)
        if (resp.status !== 200) {
          this.search_error = `(${resp.status}) ${resp.statusText}`;
          this.searching = false;
          return false;
        }
        this.search_error = null;

        // Unpackage the result.
        const resp_json = await resp.json();
        window.console.log(resp_json);

        this.query_string = resp_json.query_str;
        if (!this.agent_pairs)
          this.agent_pairs = resp_json.relations
        else
          this.agent_pairs = [...this.agent_pairs, ...resp_json.relations]
        this.next_offset = resp_json.next_offset;
        this.complexes_covered = resp_json.complexes_covered;

        this.searching = false;

        this.updateShareUrl();
        this.lastSearchOk = true;
        return true;
      },

      pushHistory: function() {
        // Handle case where we've gone back.
        if (this.history_idx !== this.search_history.length)
          this.search_history = this.search_history.slice(0, this.history_idx+1);

        // Push the latest constraint to the end of the history.
        this.search_history.push({constraints: this.deepCopy(this.constraints)});
        this.history_idx += 1;
      },

      backButton: async function() {
        if (this.cannotGoBack)
          return;
        this.history_idx -= 1;
        this.constraints = this.search_history[this.history_idx].constraints;
        this.next_offset = 0;
        this.agent_pairs = null;
        return await this.search();
      },

      forwardButton: async function() {
        if (this.cannotGoForward)
          return;
        this.history_idx += 1;
        this.constraints = this.search_history[this.history_idx].constraints;
        this.next_offset = 0;
        this.agent_pairs = null;
        return await this.search();
      },

      deepCopy: function(inObject) {
        let outObject, value, key;

        if (typeof inObject !== 'object' || inObject === null)
          return inObject;

        outObject = Array.isArray(inObject) ? [] : {};

        for (key in inObject) {
          value = inObject[key];
          outObject[key] = this.deepCopy(value);
        }

        return outObject;
      },

      getMore: async function() {
        return await this.search()
      },

      isFilled(param) {
        if (!param || !param.class || !param.constraint) return false;
        const c = param.constraint;

        if (param.class === 'HasAgent') {
          return !!c.agent_id;
        }
        if (param.class === 'HasType')   return Array.isArray(c.stmt_types) && c.stmt_types.length > 0;
        if (param.class === 'FromMeshIds') return Array.isArray(c.mesh_ids) && c.mesh_ids.length > 0;
        if (param.class === 'FromPapers')  return Array.isArray(c.paper_list) && c.paper_list.length > 0;

        return false;
      },

      validateMeshIds() {
        // Find all FromMeshIds constraints
        const meshConstraints = Object.values(this.constraints).filter(
          c => c && c.class === 'FromMeshIds' && c.constraint
        );

        for (const constraint of meshConstraints) {
          const meshIds = constraint.constraint.mesh_ids || [];

          // If mesh_ids array is empty, no validation needed (empty input is allowed)
          if (meshIds.length === 0) {
            continue;
          }

          // Validate each mesh_id
          // MeSH ID pattern: starts with a letter (typically D, C, etc.) followed by digits
          // Example: D0135456, D000086382
          const meshIdPattern = /^[A-Z]\d+$/i;

          for (const meshId of meshIds) {
            const meshIdStr = String(meshId).trim();
            // If there's a value, it must match the MeSH ID pattern
            if (meshIdStr && !meshIdPattern.test(meshIdStr)) {
              return `Invalid MeSH ID format: "${meshIdStr}". You must enter a valid MeSH ID, either directly or by searching for a term with the help of Gilda. MeSH IDs should start with the letter 'D' followed by numbers (e.g., D010612 or D000086382).`;
            }
          }
        }

        return null; // No validation errors
      },
      presetRoles(mode) {
        const agents = this.hasAgentConstraints
          .sort((a, b) => a.idx - b.idx)
          .slice(0, 2);

        if (agents.length === 0) return;

        const setRole = (pair, role) => {
          if (!pair || !pair.c || !pair.c.constraint) return;
          this.$set(pair.c.constraint, 'role', role);
        };

        if (mode === 'any-any') {
          if (agents[0]) setRole(agents[0]);
          if (agents[1]) setRole(agents[1]);
        } else if (mode === 's-o') {
          if (agents[0]) setRole(agents[0], 'subject');
          if (agents[1]) setRole(agents[1], 'object');
        } else if (mode === 'o-s') {
          if (agents[0]) setRole(agents[0], 'object');
          if (agents[1]) setRole(agents[1], 'subject');
        }
      },
       setDisplay(idx, text) {
       this.$set(this.displayTextMap, idx, (text || '').trim());
        },

       getSortedAgentPairs() {
         return this.hasAgentConstraints
           .sort((a,b) => a.idx - b.idx)
           .slice(0, 2);
       },
       _parseAgentToken(s) {
        if (!s) return { ns: 'AUTO', id: '' };
        const m = String(s).trim().match(/^(?<ns>[^:：\s]+)\s*[:：]\s*(?<id>.+)$/);
        if (m && m.groups) {
          return { ns: m.groups.ns.toUpperCase(), id: m.groups.id.trim() };
        }
        return { ns: 'AUTO', id: String(s).trim() };
       },
       async copyLink() {
          const text = this.shareUrl || window.location.href;
          try {
            if (navigator.clipboard && window.isSecureContext) {
              await navigator.clipboard.writeText(text);
              alert('Link copied:\n' + text);
            } else {
              throw new Error('no clipboard');
            }
          } catch {
            prompt('Copy this link:', text);
          }
        },

      buildReadableParams() {
        const params = new URLSearchParams();
        const names = ['agent','other_agent'];
          const pairs = this.getSortedAgentPairs();
         pairs.forEach((pair, i) => {
           const a = pair.c?.constraint || {};
           if (!a.agent_id) return;
           const display = (this.displayTextMap[pair.idx] || '').trim();
           const token = display || ((a.namespace && a.namespace !== 'AUTO')
             ? `${String(a.namespace).toLowerCase()}:${a.agent_id}`
             : a.agent_id);
           params.set(names[i], token);
           if (a.role && a.role !== 'any') params.set(`role${i+1}`, a.role);
         });

        const typePair = this.nonAgentConstraints.find(p => p.c.class === 'HasType');
        const stmtTypes = typePair?.c?.constraint?.stmt_types || [];
        if (stmtTypes.length) params.set('rel', stmtTypes.join(','));

        const meshPair = this.nonAgentConstraints.find(p => p.c.class === 'FromMeshIds');
        const meshIds = meshPair?.c?.constraint?.mesh_ids || [];
        if (meshIds.length) params.set('mesh', meshIds.join(','));

        const paperPair = this.nonAgentConstraints.find(p => p.c.class === 'FromPapers');
        const paperList = paperPair?.c?.constraint?.paper_list || [];
        if (paperList.length) {
          const items = paperList.map(([id_type, paper_id]) => `${paper_id}@${id_type}`);
          params.set('papers', items.join(','));
        }

        params.set('autosearch', '1');
        return params;
      },

      updateShareUrl() {
        const params = this.buildReadableParams();
        // Defensive check: ensure window.location.href is valid before constructing URL
        const baseUrl = window.location.href || window.location.origin + window.location.pathname;
        if (!baseUrl || baseUrl === '') {
          console.warn('Cannot update share URL: window.location.href is empty');
          return;
        }
        try {
          const url = new URL(baseUrl);
          url.search = params.toString();
          history.replaceState(null, '', url.toString());
          this.shareUrl = url.toString();
        } catch (e) {
          console.warn('Failed to construct URL for share link:', e);
        }
      },

      applyReadableParams(params) {
        this.constraints = {};
        this.cidx = 0;

        this.addConstraint('HasAgent');
        this.addConstraint('HasAgent');
        this.addConstraint('HasType');
        this.addConstraint('FromMeshIds');

        const [a1, a2] = this.hasAgentConstraints
          .sort((x, y) => x.idx - y.idx)
          .slice(0, 2);
       const raw1 = params.get('agent') || '';
       const raw2 = params.get('other_agent') || '';
        // 3) fill Agent
       const t1 = this._parseAgentToken(raw1);
       const t2 = this._parseAgentToken(raw2);

      if (a1 && t1.id) {
        this.$set(a1.c, 'constraint', {
          ...(a1.c.constraint || {}),
          agent_id: t1.id,
          namespace: t1.ns,
        });
        this.$set(this.displayTextMap, a1.idx, raw1);
      }
      if (a2 && t2.id) {
        this.$set(a2.c, 'constraint', {
          ...(a2.c.constraint || {}),
          agent_id: t2.id,
          namespace: t2.ns,
        });
        this.$set(this.displayTextMap, a2.idx, raw2);
      }


      //role
      const role1 = params.get('role1');
      const role2 = params.get('role2');
      if (a1 && role1 && role1 !== 'any') this.$set(a1.c.constraint, 'role', role1);
      if (a2 && role2 && role2 !== 'any') this.$set(a2.c.constraint, 'role', role2);

      //rel
      const rel = params.get('rel');
      if (rel) {
        const arr = rel.split(',').map(s => s.trim()).filter(Boolean);
        const typePair = this.nonAgentConstraints.find(p => p.c.class === 'HasType');
        if (typePair) this.$set(typePair.c, 'constraint', { stmt_types: arr });
      }

      //MeSH
      const mesh = params.get('mesh');
      if (mesh) {
        const arr = mesh.split(',').map(s => s.trim()).filter(Boolean);
        const m = this.nonAgentConstraints.find(p => p.c.class === 'FromMeshIds');
        this.$set(m.c, 'constraint', { mesh_ids: arr });
      }

      //Papers
      const papers = params.get('papers');
      if (papers) {
        this.addConstraint('FromPapers');
        const arr = papers
          .split(',')
          .map(s => s.trim())
          .filter(Boolean)
          .map(tok => {
            if (tok.includes('@')) {
              const [pid, it] = tok.split('@');
              return [it.toLowerCase(), pid];
            }
            if (tok.includes(':')) {
              const [it, pid] = tok.split(':');
              return [it.toLowerCase(), pid];
            }
            return ['pmid', tok];
          });
        const p = this.nonAgentConstraints.find(p => p.c.class === 'FromPapers');
        this.$set(p.c, 'constraint', { paper_list: arr });
      }

      this.$nextTick(() => { this.exampleTick++; });
      this.ensureBlankSlot();
    },

    async applyUrlParamsAndSearch() {
      const params = new URLSearchParams(window.location.search);
      const hasReadable =
        params.get('agent') || params.get('other_agent') ||
        params.get('rel') || params.get('mesh') || params.get('papers');

      const autosearch = params.get('autosearch') === '1';

      if (hasReadable) {
        this.applyReadableParams(params);
      } else {
        return;
      }

      if (autosearch) {
        this.next_offset = 0;
        this.agent_pairs = null;
        await this.search();
      }
    },
    },
    computed: {
      empty_relations: function() {
        if (this.agent_pairs === null)
          return true;
        return (this.agent_pairs.length === 0);
      },

      base_list: function() {
        return this.agent_pairs;
      },

      cannotGoBack: function() {
        return this.history_idx <= 0;
      },

      cannotGoForward: function() {
        return this.history_idx >= (this.search_history.length - 1);
      },
      hasAgentConstraints () {
        return Object.keys(this.constraints)
          .map(k => ({ idx: Number(k), c: this.constraints[k] }))
          .filter(pair => pair.c && pair.c.class === 'HasAgent');
      },

      nonAgentConstraints () {
        return Object.keys(this.constraints)
          .map(k => ({ idx: Number(k), c: this.constraints[k] }))
          .filter(pair => pair.c && pair.c.class !== 'HasAgent');
      },
      currentPreset () {
        const agents = [...this.hasAgentConstraints]
          .sort((a,b) => a.idx - b.idx)
          .slice(0, 2);

        const roleOf = (i) =>
          agents[i] && agents[i].c && agents[i].c.constraint
            ? (agents[i].c.constraint.role || 'any')
            : 'any';

        const r1 = roleOf(0);
        const r2 = roleOf(1);
        if (r1 === 'subject' && r2 === 'object') return 's-o';
        if (r1 === 'object' && r2 === 'subject') return 'o-s';
        if (r1 === 'subject' && (r2 === 'any')) return 's-o';
        if (r1 === 'object' && (r2 === 'any')) return 'o-s';
        if (r1 === 'any' && r2 === 'subject') return 'o-s';
        if (r1 === 'any' && r2 === 'object') return 's-o';
        if (r1 === 'any' && r2 === 'any') return 'any-any';
        return 'any-any';
    }

    },
    created() {
      this.addConstraint('HasAgent');
      this.addConstraint('HasAgent');
      this.addConstraint('HasType');
      this.addConstraint('FromMeshIds');
      this.ensureBlankSlot();
    },
   mounted() {
      this.applyUrlParamsAndSearch();

    this._onExample = (e) => {
      const d = e.detail || {};
      if (this.hasAgentConstraints.length < 2) {
        this.addConstraint('HasAgent');
      }

      const a1 = this.hasAgentConstraints.sort((a,b)=>a.idx-b.idx)[0];
      const a2 = this.hasAgentConstraints.sort((a,b)=>a.idx-b.idx)[1];

      if (a1) {
        this.$set(a1.c, 'constraint', { ...(a1.c.constraint || {}),
          agent_id: d.agent1 || '' ,
          namespace: 'AUTO'
        });
      }
      if (a2) {
        this.$set(a2.c, 'constraint', { ...(a2.c.constraint || {}),
          agent_id: d.agent2 || '' ,
          namespace: 'AUTO'
        });
      }

      // set role setting by role
      // In exmaple text, use data-preset="s-o" / "o-s" / "any-any"
      const preset = d.preset || (d.role1 === 'subject' && d.role2 === 'object'
        ? 's-o'
        : d.role1 === 'object' && d.role2 === 'subject'
        ? 'o-s'
        : 'any-any');
      this.presetRoles(preset);

      const typePair = this.nonAgentConstraints.find(p => p.c.class === 'HasType');

      if (d.stmtType) {
        if (typePair) {
          this.$set(typePair.c, 'constraint', { stmt_types: [d.stmtType] });
        } else {
          this.addConstraint('HasType');
          const last = Math.max(...Object.keys(this.constraints).map(Number));
          this.$set(this.constraints[last], 'constraint', { stmt_types: [d.stmtType] });
        }
      } else if (typePair) {
        this.$set(typePair.c, 'constraint', { stmt_types: [] });
      }

      // Clear mesh constraint when example is clicked
      const meshPair = this.nonAgentConstraints.find(p => p.c.class === 'FromMeshIds');
      if (meshPair) {
        this.$set(meshPair.c, 'constraint', { mesh_ids: [] });
      }

    this.$nextTick(() => { this.exampleTick++; });
    };

    window.addEventListener('indra:example', this._onExample);
    },

    beforeDestroy() {
        window.removeEventListener('indra:example', this._onExample);
    },

    mixins: [piecemeal_mixin]
  }
</script>

<style scoped>
  .relation-search {
    cursor: pointer
  }

  select, input, button {
    margin: 0.2em;
  }
  .spaced {
    margin: 0 0.5em;
  }

  .nav-btn {
    margin-top: auto;
    margin-bottom: auto;
  }

  #result-list {
    margin-top: 10px;
  }
  .role-presets {
  display: flex;
  }

  .btn-role {
    background-color: #fff;
    border: 1px solid #ddd;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 120px;
    cursor: pointer;
    border-radius: 6px;
  }

  .btn-role img {
    max-width: 95%;
    max-height: 100%;
    margin-top: 6px;
  }

  .role-text {
    font-size: 12px;
    margin-top: 4px;
  }

  .btn-role.active {
    background-color: #007bff;
    color: #fff;
  }
    .blue-dot {
      background-color: #6fa8dcff;
  }
  .orange-dot {

      background-color: #ff9900ff;

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

  .form-row {
    margin-bottom: 15px;
    border-style: solid;
    border-color: rgba(209, 209, 209, 1);
    border-image: none;
    border-width: 2px;
    border-radius: 5px;
    border-top-left-radius: 5px;
    border-top-right-radius: 5px;
    border-bottom-right-radius: 5px;
    border-bottom-left-radius: 5px;
  }

</style>
