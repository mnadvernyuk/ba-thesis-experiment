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

        <p><strong>For each story, you will have to do two tasks:</strong></p>
        <ol>
          <li>Write a brief explanation of why you think the outcome happened.</li>
          <li>Select the <em>principal cause</em> of the outcome from four options.</li>
        </ol>

        <p>
          There are no right or wrong answers, please respond with your best judgment.
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

            <p><strong>Event A:</strong> {{ trial.event_A }}</p>
            <p><strong>Event B:</strong> {{ trial.event_B }}</p>
            <p><strong>Event C:</strong> {{ trial.event_C }}</p>
            <p><strong>Event D:</strong> {{ trial.event_D }}</p>

            <p class="outcome">
              <strong>Outcome:</strong> {{ trial.outcome_shown }}
            </p>
          </div>
        </InstructionScreen>

        <!-- Free-text explanation -->
        <InstructionScreen :key="`why-${idx}`" :title="'Your explanation'">
          <div class="block">
            <p><strong>Question:</strong> Why did the outcome happen?</p>

            <!-- textarea -->
            <textarea
              v-model="responses[idx].freeText"
              rows="5"
              class="textarea"
              placeholder="Type your explanation here…"
            ></textarea>

            <p class="hint">Please write at least one sentence.</p>
          </div>
        </InstructionScreen>

        <!-- Cause choice -->
        <InstructionScreen :key="`cause-${idx}`" :title="'Select the principal cause'">
          <div class="block">
            <p>
              From the four events below, please select the <strong>principal cause</strong>
              of the outcome.
            </p>

            <label class="choice">
              <input type="radio" :name="`cause-${idx}`" value="A" v-model="responses[idx].causeChoice" />
              <strong>A:</strong> {{ trial.event_A }}
            </label>

            <label class="choice">
              <input type="radio" :name="`cause-${idx}`" value="B" v-model="responses[idx].causeChoice" />
              <strong>B:</strong> {{ trial.event_B }}
            </label>

            <label class="choice">
              <input type="radio" :name="`cause-${idx}`" value="C" v-model="responses[idx].causeChoice" />
              <strong>C:</strong> {{ trial.event_C }}
            </label>

            <label class="choice">
              <input type="radio" :name="`cause-${idx}`" value="D" v-model="responses[idx].causeChoice" />
              <strong>D:</strong> {{ trial.event_D }}
            </label>

            <p class="hint">
              (If you are unsure, please choose the option that seems most important.)
            </p>
          </div>
        </InstructionScreen>
      </template>

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
      responses: []
    };
  },
  methods: {
    // Option 1: pretty title from CSV, no counters at all
    screenTitle(trial) {
      return trial.scenario_title || "Story";
    },
    // Stable ID for counterbalancing (good for prof; later can replace with Prolific PID)
    getParticipantId() {
      const key = "ba_thesis_pid";
      let pid = localStorage.getItem(key);
      if (!pid) {
        pid = `${Date.now()}_${Math.random().toString(16).slice(2)}`;
        localStorage.setItem(key, pid);
      }
      return pid;
    }
  },
  async mounted() {
    try {
      const res = await fetch("stimuli/vignettes.csv");
      if (!res.ok) throw new Error(`Failed to fetch CSV: ${res.status}`);

      const csvText = await res.text();
      const parsed = Papa.parse(csvText, { header: true, skipEmptyLines: true });

      const rows = parsed.data.filter((r) => r.scenario_id);
      if (!rows.length) {
        throw new Error("CSV loaded but has no rows with scenario_id.");
      }

      // normal order as in csv
      const scenarioOrder = [];
      const seen = new Set();
      for (const r of rows) {
        if (!seen.has(r.scenario_id)) {
          seen.add(r.scenario_id);
          scenarioOrder.push(r.scenario_id);
        }
      }

      // Group rows by scenario_id
      const byScenario = rows.reduce((acc, row) => {
        (acc[row.scenario_id] ||= []).push(row);
        return acc;
      }, {});

      // Counterbalanced outcomes per participant: half negative, half positive (flipped between variants)
      const pid = this.getParticipantId();
      const variant = pid.charCodeAt(0) % 2; // 0 or 1

      const n = scenarioOrder.length;
      const half = Math.floor(n / 2);

      const valencePattern =
        variant === 0
          ? Array.from({ length: n }, (_, i) => (i < half ? "negative" : "positive"))
          : Array.from({ length: n }, (_, i) => (i < half ? "positive" : "negative"));

      // Distal type selection: stable (not random) across reloads for this participant.
      this.trials = scenarioOrder.map((sid, i) => {
        const candidates = byScenario[sid];
        if (!candidates || !candidates.length) {
          throw new Error(`No rows found for scenario_id="${sid}"`);
        }

        // stable pick: depends on variant + scenario index
        const chosen = candidates[(variant + i) % candidates.length];

        const valence = valencePattern[i];
        const outcomeShown =
          valence === "negative" ? chosen.outcome_negative : chosen.outcome_positive;

        return { ...chosen, valence, outcome_shown: outcomeShown };
      });

      // Preparing response objects (one per trial)
      this.responses = this.trials.map((t) => ({
        scenario_id: t.scenario_id,
        scenario_title: t.scenario_title,
        distal_type: t.distal_type,
        valence: t.valence,
        freeText: "",
        causeChoice: ""
      }));

      console.log("Loaded trials:", this.trials);
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
/* Consistent typography / layout */
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

.textarea {
  width: 100%;
  max-width: 720px;
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
</style>
