<template>
  <Experiment title="BA Thesis Experiment">
    <!-- Welcome -->
    <InstructionScreen :title="'Welcome!'">
      <div class="block">
        <p>
          Thank you for taking part in the experiment.
          In this study you will read a series of short stories describing chains of events.
          Each story ends in an outcome.
        </p>

        <p><strong>For each story, you will do one task:</strong></p>
        <ol>
          <li>
            Select the event that seems like the <em>best candidate</em> for saying that it caused the outcome.
          </li>
        </ol>

        <p>
          There are no right or wrong answers — please respond with your best judgment.
        </p>

        <p><em>Click “Next” to begin with the first story.</em></p>
      </div>
    </InstructionScreen>

    <!-- Loading -->
    <InstructionScreen v-if="loading" :title="'Loading…'">
      <div class="block">Preparing the experiment.</div>
    </InstructionScreen>

    <!-- Error screen -->
    <InstructionScreen v-else-if="errorMessage" :title="'Error'">
      <div class="block">
        <p><strong>Something went wrong:</strong></p>
        <p>{{ errorMessage }}</p>
        <p class="hint">
          Check that the CSV is at <code>public/stimuli/vignettes.csv</code> and that images are in
          <code>public/images/</code>.
        </p>
      </div>
    </InstructionScreen>

    <!-- Trials -->
    <template v-else>
      <template v-for="(trial, idx) in trials">
        <!-- Story screen -->
        <InstructionScreen :key="`story-${idx}`" :title="screenTitle(trial)">
          <div class="block">
            <div class="imageWrap" v-if="trial.image">
              <img :src="trial.image" alt="Scenario illustration" class="stimImage" />
            </div>

            <p class="readCarefully">Please read the following story carefully.</p>

            <p><strong>Event A:</strong> {{ trial.event_A }}</p>
            <p><strong>Event B:</strong> {{ trial.event_B }}</p>
            <p><strong>Event C:</strong> {{ trial.event_C }}</p>
            <p><strong>Event D:</strong> {{ trial.event_D }}</p>

            <p class="outcome">
              <strong>Outcome:</strong> {{ trial.outcome_shown }}
            </p>
          </div>
        </InstructionScreen>

        <!-- Cause choice screen -->
        <InstructionScreen :key="`cause-${idx}`" :title="'Which event best caused the outcome?'">
          <div class="block">
            <p>
              From the four events below, please select the event that seems like the
              <strong>best candidate</strong> for saying that it caused the outcome.
            </p>

            <label class="choice">
              <input
                type="radio"
                :name="`cause-${idx}`"
                value="A"
                v-model="responses[idx].causeChoice"
              />
              <strong>A:</strong> {{ trial.event_A }}
            </label>

            <label class="choice">
              <input
                type="radio"
                :name="`cause-${idx}`"
                value="B"
                v-model="responses[idx].causeChoice"
              />
              <strong>B:</strong> {{ trial.event_B }}
            </label>

            <label class="choice">
              <input
                type="radio"
                :name="`cause-${idx}`"
                value="C"
                v-model="responses[idx].causeChoice"
              />
              <strong>C:</strong> {{ trial.event_C }}
            </label>

            <label class="choice">
              <input
                type="radio"
                :name="`cause-${idx}`"
                value="D"
                v-model="responses[idx].causeChoice"
              />
              <strong>D:</strong> {{ trial.event_D }}
            </label>

            <p class="hint">
              (If you are unsure, please choose the option that seems most important.)
            </p>
          </div>
        </InstructionScreen>

        <!-- One attention check mid-way -->
        <InstructionScreen
          v-if="idx === attentionIndex"
          :key="`attn-${idx}`"
          :title="'Attention check'"
        >
          <div class="block">
            <p>
              To show that you are paying attention, please select <strong>Option C</strong> below.
            </p>

            <label class="choice">
              <input type="radio" :name="'attn-check'" value="A" v-model="attentionCheck" />
              Option A
            </label>
            <label class="choice">
              <input type="radio" :name="'attn-check'" value="B" v-model="attentionCheck" />
              Option B
            </label>
            <label class="choice">
              <input type="radio" :name="'attn-check'" value="C" v-model="attentionCheck" />
              Option C
            </label>
            <label class="choice">
              <input type="radio" :name="'attn-check'" value="D" v-model="attentionCheck" />
              Option D
            </label>

            <p class="hint">
              (This is just a check. There are no “right answers” in the main task.)
            </p>
          </div>
        </InstructionScreen>
      </template>

      <!-- NEW: explicit feedback explanation BEFORE PostTestScreen -->
      <InstructionScreen :title="'Final feedback (optional)'">
        <div class="block">
          <p>
            Before finishing, you can leave optional feedback in the next screen.
          </p>
          <ul>
            <li>Anything unclear or confusing?</li>
            <li>Any scenario that felt odd, unrealistic, or harder to understand?</li>
            <li>Any technical issues (images not loading, layout problems, etc.)?</li>
            <li>Any other concerns or suggestions?</li>
          </ul>
          <p class="hint">
            This feedback helps improve the experiment. You can also leave it blank.
          </p>
        </div>
      </InstructionScreen>

      <!-- Default magpie post-test screen (includes comment box) -->
      <PostTestScreen />

      <!-- End -->
      <InstructionScreen :title="'Thank you!'">
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
      attentionCheck: "",
      attentionIndex: 0
    };
  },
  methods: {
    screenTitle(trial) {
      return trial.scenario_title || "Story";
    },

    // Use Prolific PID if available; otherwise generate stable local id
    getParticipantId() {
      const url = new URL(window.location.href);
      const prolific = url.searchParams.get("PROLIFIC_PID");
      if (prolific) return prolific;

      const key = "ba_thesis_pid";
      let pid = localStorage.getItem(key);
      if (!pid) {
        pid = `${Date.now()}_${Math.random().toString(16).slice(2)}`;
        localStorage.setItem(key, pid);
      }
      return pid;
    },

    // deterministic hash -> int
    hashStringToInt(str) {
      let h = 2166136261;
      for (let i = 0; i < str.length; i++) {
        h ^= str.charCodeAt(i);
        h = Math.imul(h, 16777619);
      }
      return h >>> 0;
    },

    // seeded RNG
    mulberry32(seed) {
      return function () {
        let t = (seed += 0x6d2b79f5);
        t = Math.imul(t ^ (t >>> 15), t | 1);
        t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
        return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
      };
    },

    shuffleInPlace(arr, rand) {
      for (let i = arr.length - 1; i > 0; i--) {
        const j = Math.floor(rand() * (i + 1));
        [arr[i], arr[j]] = [arr[j], arr[i]];
      }
      return arr;
    },

    // Create a balanced-ish distal array for ANY number of trials
    makeDistalArray(n) {
      const base = ["natural", "non_deliberate", "deliberate"];
      const out = [];
      for (let i = 0; i < n; i++) out.push(base[i % base.length]);
      return out;
    },

    // Create a half/half valence array for ANY n (difference at most 1)
    makeValenceArray(n) {
      const half = Math.floor(n / 2);
      const out = [];
      for (let i = 0; i < n; i++) out.push(i < half ? "negative" : "positive");
      return out;
    }
  },

  async mounted() {
    try {
      const res = await fetch("stimuli/vignettes.csv");
      if (!res.ok) throw new Error(`Failed to fetch CSV: ${res.status}`);

      const csvText = await res.text();
      const parsed = Papa.parse(csvText, { header: true, skipEmptyLines: true });

      const rows = parsed.data.filter((r) => r.scenario_id);
      if (!rows.length) throw new Error("CSV loaded but has no rows with scenario_id.");

      // unique scenario list (use ALL scenarios in CSV)
      const scenarioIds = [];
      const seen = new Set();
      for (const r of rows) {
        if (!seen.has(r.scenario_id)) {
          seen.add(r.scenario_id);
          scenarioIds.push(r.scenario_id);
        }
      }

      // group by scenario_id
      const byScenario = rows.reduce((acc, row) => {
        (acc[row.scenario_id] ||= []).push(row);
        return acc;
      }, {});

      const pid = this.getParticipantId();
      const rand = this.mulberry32(this.hashStringToInt(pid));

      const n = scenarioIds.length;

      // build arrays and shuffle independently (per prof)
      const scenarioArr = [...scenarioIds];
      const distalArr = this.makeDistalArray(n);
      const valenceArr = this.makeValenceArray(n);

      this.shuffleInPlace(scenarioArr, rand);
      this.shuffleInPlace(distalArr, rand);
      this.shuffleInPlace(valenceArr, rand);

      // randomize trial order entirely as well (already accomplished via scenarioArr shuffle)
      // set attention check around the middle
      this.attentionIndex = Math.max(0, Math.floor(n / 2) - 1);

      this.trials = scenarioArr.map((sid, i) => {
        const candidates = byScenario[sid] || [];
        if (!candidates.length) throw new Error(`No rows found for scenario_id="${sid}"`);

        const wantedDistal = distalArr[i];
        const chosenRow =
          candidates.find((c) => (c.distal_type || "").trim() === wantedDistal) || candidates[0];

        const valence = valenceArr[i];
        const outcomeShown =
          valence === "negative" ? chosenRow.outcome_negative : chosenRow.outcome_positive;

        return { ...chosenRow, valence, outcome_shown: outcomeShown };
      });

      this.responses = this.trials.map((t) => ({
        scenario_id: t.scenario_id,
        scenario_title: t.scenario_title,
        distal_type: t.distal_type,
        valence: t.valence,
        causeChoice: ""
      }));

      console.log("PID:", pid);
      console.log("ASSIGNMENTS:", this.trials.map((t) => [t.scenario_id, t.distal_type, t.valence]));
    } catch (err) {
      console.error(err);
      this.errorMessage = String(err.message || err);
    } finally {
      this.loading = false;
    }
  }
};
</script>

<style scoped>
.block {
  max-width: 820px;
  margin: 0 auto;
}

.imageWrap {
  text-align: center;
  margin-bottom: 16px;
}

.stimImage {
  max-width: 520px;
  width: 100%;
  height: auto;
  border-radius: 10px;
}

.choice {
  display: block;
  margin: 10px 0;
}

.outcome {
  margin-top: 14px;
}

.hint {
  margin-top: 10px;
  font-size: 0.95em;
  color: #555;
}

.readCarefully {
  margin: 6px 0 14px 0;
  font-size: 0.95em;
  color: #777;
}
</style>