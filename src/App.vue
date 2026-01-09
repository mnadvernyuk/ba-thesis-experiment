<template>
  <Experiment title="BA Thesis Experiment">
    <!-- Welcome -->
    <InstructionScreen title="Welcome!">
      <div class="block">
        <p>
          Thank you for taking part in the experiment.
          You will read a series of short stories describing chains of events.
          Each story ends in an outcome.
        </p>

        <p>
          <strong>Your task is the following:</strong>
          For each story, please select
          <strong>the event that seems like the best candidate for saying that it caused the outcome</strong>.
        </p>

        <p>There are no right or wrong answers. Please respond with your best judgment.</p>
        <p><em>Click “Next” to begin with the first story.</em></p>
      </div>
    </InstructionScreen>

    <!-- Loading -->
    <InstructionScreen v-if="loading" title="Loading…">
      <div class="block">Preparing the experiment.</div>
    </InstructionScreen>

    <!-- Error -->
    <InstructionScreen v-else-if="errorMessage" title="Error">
      <div class="block">
        <p><strong>Something went wrong:</strong></p>
        <p>{{ errorMessage }}</p>
        <p class="hint">
          Make sure the CSV is at <code>public/stimuli/vignettes.csv</code> and images are in
          <code>public/images/</code>. Image paths in the CSV should look like <code>images/xyz.png</code>.
        </p>
      </div>
    </InstructionScreen>

    <!-- Trials -->
    <template v-else>
      <InstructionScreen
        v-for="(trial, idx) in trials"
        :key="`trial-${idx}-${trial.scenario_id}`"
        :title="trial.scenario_title || 'Story'"
      >
        <div class="block">
          <div v-if="trial.image" class="imageWrap">
            <img :src="trial.image" class="stimImage" alt="Scenario illustration" />
          </div>

          <p class="readCarefully">
            Please read the following story carefully.
            <span class="taskLine">
              Then select the event that seems like the best candidate for saying that it caused the outcome.
            </span>
          </p>

          <label class="choice">
            <input
              type="radio"
              :name="`cause-${idx}`"
              value="A"
              v-model="responses[idx].cause_choice"
              @change="blockedIdx = null"
            />
            <strong>Event A:</strong> {{ trial.event_A }}
          </label>

          <label class="choice">
            <input
              type="radio"
              :name="`cause-${idx}`"
              value="B"
              v-model="responses[idx].cause_choice"
              @change="blockedIdx = null"
            />
            <strong>Event B:</strong> {{ trial.event_B }}
          </label>

          <label class="choice">
            <input
              type="radio"
              :name="`cause-${idx}`"
              value="C"
              v-model="responses[idx].cause_choice"
              @change="blockedIdx = null"
            />
            <strong>Event C:</strong> {{ trial.event_C }}
          </label>

          <label class="choice">
            <input
              type="radio"
              :name="`cause-${idx}`"
              value="D"
              v-model="responses[idx].cause_choice"
              @change="blockedIdx = null"
            />
            <strong>Event D:</strong> {{ trial.event_D }}
          </label>

          <p class="outcome">
            <strong>Outcome:</strong> {{ trial.outcome_shown }}
          </p>

          <p class="hint">(If you are unsure, please choose the option that seems most important.)</p>

          <p v-if="blockedIdx === idx" class="pleaseAnswer">
            Please select an event before continuing.
          </p>
        </div>
      </InstructionScreen>

      <PostTestScreen />

      <InstructionScreen title="Thank you!">
        <div class="block">
          <p>You have completed the study.</p>
        </div>
      </InstructionScreen>

      <SubmitResultsScreen />
    </template>
  </Experiment>
</template>

<script>
import Papa from "papaparse";

export default {
  name: "App",

  data() {
    return {
      loading: true,
      errorMessage: "",
      trials: [],
      responses: [],
      blockedIdx: null,
      _nextGateHandler: null,
      _unwatchMagpie: null,
      _pid: ""
    };
  },

  methods: {
    getParticipantId() {
      const key = "ba_thesis_pid";
      let pid = localStorage.getItem(key);
      if (!pid) {
        pid = `${Date.now()}_${Math.random().toString(16).slice(2)}`;
        localStorage.setItem(key, pid);
      }
      return pid;
    },

    seedFromPid(pid) {
      let h = 0;
      for (let i = 0; i < pid.length; i++) h = (h * 31 + pid.charCodeAt(i)) >>> 0;
      return h;
    },

    shuffle(arr, seed) {
      const a = arr.slice();
      let s = seed >>> 0;
      const rand = () => {
        s = (1664525 * s + 1013904223) >>> 0;
        return s / 4294967296;
      };
      for (let i = a.length - 1; i > 0; i--) {
        const j = Math.floor(rand() * (i + 1));
        [a[i], a[j]] = [a[j], a[i]];
      }
      return a;
    },

    isAttentionRow(row) {
      const raw = row?.is_attention_check;
      if (raw === undefined || raw === null) return false;
      const s = String(raw).trim().toLowerCase();
      if (s === "1" || s === "true" || s === "yes") return true;
      const m = s.match(/\d+/);
      return m ? Number(m[0]) === 1 : false;
    },

    normDistal(s) {
      const t = String(s ?? "").trim().toLowerCase();
      if (!t) return "";
      if (t === "non-deliberate" || t === "nondeliberate" || t === "non deliberate") return "non_deliberate";
      return t;
    },

    scenarioIdFromLabel(label) {
      const l = String(label ?? "").trim().toLowerCase();
      const map = {
        car: "car_accident",
        memory: "memory_test",
        theatre: "theater_interruption",
        theater: "theater_interruption",
        metro: "metro_escalator",
        airport: "airport_luggage",
        apartment: "apartment_explosion"
      };
      return map[l] || "";
    },

    getVisibleTrialIdx() {
      const inputs = Array.from(document.querySelectorAll('input[type="radio"][name^="cause-"]'));
      const visible = inputs.find(el => el.offsetParent !== null);
      if (!visible) return null;
      const m = String(visible.name).match(/^cause-(\d+)$/);
      return m ? Number(m[1]) : null;
    },

    installNextGate() {
      if (this._nextGateHandler) return;

      this._nextGateHandler = (e) => {
        const t = e.target;
        if (!t) return;

        const text = (t.innerText || t.value || "").trim().toLowerCase();
        if (text !== "next") return;

        const idx = this.getVisibleTrialIdx();
        if (idx === null) return;

        const hasChoice = !!this.responses?.[idx]?.cause_choice;
        if (hasChoice) return;

        e.preventDefault();
        e.stopPropagation();
        if (typeof e.stopImmediatePropagation === "function") e.stopImmediatePropagation();

        this.blockedIdx = idx;
        window.setTimeout(() => {
          if (this.blockedIdx === idx) this.blockedIdx = null;
        }, 1200);
      };

      document.addEventListener("click", this._nextGateHandler, true);
    },

    buildTrialRows() {
      const pid = this._pid || "";
      return (this.trials || []).map((t, i) => {
        const r = (this.responses || [])[i] || {};
        return {
          pid,
          trial_index: i,
          scenario_id: t?.scenario_id || "",
          scenario_title: t?.scenario_title || "",
          distal_type: t?.distal_type || "",
          valence: t?.valence || "",
          outcome_shown: t?.outcome_shown || "",
          is_attention_check: this.isAttentionRow(t) ? 1 : 0,
          attention_correct: r?.attention_correct || "",
          cause_choice: r?.cause_choice || ""
        };
      });
    },

    installMagpieMeasurementsAsArray() {
      if (!this.$magpie) {
        console.warn("Magpie instance not found on this.$magpie.");
        return;
      }

      const write = () => {
        const rows = this.buildTrialRows();
        this.$magpie.measurements = rows; // <-- CRUCIAL: array goes here
      };

      write();

      if (this._unwatchMagpie) this._unwatchMagpie();
      this._unwatchMagpie = this.$watch("responses", write, { deep: true });
    }
  },

  async mounted() {
    try {
      const res = await fetch("stimuli/vignettes.csv");
      if (!res.ok) throw new Error(`CSV load failed: ${res.status}`);
      const csv = await res.text();

      const parsed = Papa.parse(csv, { header: true, skipEmptyLines: true });
      const rows = (parsed.data || []).filter(r => r && r.scenario_id);

      if (!rows.length) throw new Error("No valid rows in CSV (missing scenario_id).");

      const attentionRow = rows.find(r => this.isAttentionRow(r)) || null;
      const realRows = rows.filter(r => !this.isAttentionRow(r));

      for (const r of realRows) r.distal_type = this.normDistal(r.distal_type);

      const profScenarioLabels = ["car", "memory", "theatre", "metro", "airport", "apartment"];
      const profScenarioIds = profScenarioLabels.map(l => this.scenarioIdFromLabel(l)).filter(Boolean);

      const byScenario = realRows.reduce((acc, r) => {
        (acc[r.scenario_id] ||= []).push(r);
        return acc;
      }, {});

      const pid = this.getParticipantId();
      this._pid = pid;
      const seed = this.seedFromPid(pid);

      const scenarioArr = this.shuffle(profScenarioIds, seed ^ 0x111);

      const distalBase = ["natural", "non_deliberate", "deliberate", "natural", "non_deliberate", "deliberate"];
      const valenceBase = ["positive", "negative", "positive", "negative", "positive", "negative"];

      const distals = this.shuffle(distalBase, seed ^ 0x222);
      const valences = this.shuffle(valenceBase, seed ^ 0x333);

      const assigned = scenarioArr.map((sid, i) => {
        const candidates = byScenario[sid] || [];
        const assignedDistal = distals[i];

        const chosen =
          candidates.find(c => this.normDistal(c.distal_type) === assignedDistal) || candidates[0] || {};

        const val = valences[i];
        const outcomeShown = val === "positive" ? chosen.outcome_positive : chosen.outcome_negative;

        return {
          ...chosen,
          distal_type: this.normDistal(chosen.distal_type),
          valence: val,
          outcome_shown: outcomeShown
        };
      });

      const builtReal = this.shuffle(assigned, seed ^ 0x444);

      const builtAll = builtReal.slice();
      if (attentionRow) {
        builtAll.splice(3, 0, {
          ...attentionRow,
          outcome_shown: attentionRow.outcome_positive || attentionRow.outcome_negative || ""
        });
      }

      this.trials = builtAll;

      this.responses = this.trials.map(t => ({
        scenario_id: t.scenario_id,
        is_attention_check: this.isAttentionRow(t),
        attention_correct: t.attention_correct || "",
        cause_choice: ""
      }));

      this.$nextTick(() => this.installNextGate());
      this.installMagpieMeasurementsAsArray();
    } catch (err) {
      console.error(err);
      this.errorMessage = err?.message || String(err);
    } finally {
      this.loading = false;
    }
  },

  beforeDestroy() {
    if (this._unwatchMagpie) this._unwatchMagpie();
    if (this._nextGateHandler) document.removeEventListener("click", this._nextGateHandler, true);
  },

  unmounted() {
    if (this._unwatchMagpie) this._unwatchMagpie();
    if (this._nextGateHandler) document.removeEventListener("click", this._nextGateHandler, true);
  }
};
</script>

<style scoped>
.block { max-width: 820px; margin: 0 auto; }

.imageWrap { text-align: center; margin-bottom: 16px; }

.stimImage { max-width: 520px; width: 100%; border-radius: 10px; }

.readCarefully { color: #777; margin-bottom: 12px; line-height: 1.35; }

.taskLine { display: block; margin-top: 6px; color: #666; }

.choice { display: block; margin: 10px 0; }

.outcome { margin-top: 16px; }

.hint { font-size: 0.95em; color: #555; }

.pleaseAnswer { margin-top: 8px; color: #b00020; font-size: 0.95em; }
</style>
