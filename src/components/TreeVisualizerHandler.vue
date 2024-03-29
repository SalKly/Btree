<template>
  <div
    id="tree-visualizer-handler"
    class="flex tree-container"
  >
    <div class="title">
      <h1 class="name">
        Structure: {{ title }}
      </h1>
    </div>
    <div class="flex buttons">
      <InsertInput
        :disabled="isAnimating"
        @myEvent="insertInputEvent"
      />
      <SearchInput
        :disabled="isAnimating"
        @myEvent="SetSearchValue"
      />

      <DeleteInput
        :disabled="isAnimating"
        :onlypop="onlypop"
        @myEvent="deleteInputEvent"
      />
    </div>

    <h1 :class="{'hidden':Found}">
      Value {{ searchValueData }} not found
    </h1>

    <Visualizer
      :sequences="sequences"
      :current-sequence-number="currentSequenceNumber"
      :current-frame-number="currentFrame"
      :tree-type="$route.params.code"
    />
    <div class="arrow-button-container flex mx-auto my-2">
      <div class="history-btn-container">
        <HistoryButtons
          id="history.buttons"
          class="mx-4"
          :current="currentSequenceNumber"
          :length="sequences.length"
          :disabled="historyButtonsAreDisabled"
          label="History"
          @HistoryEvents="changeCurrentSequence"
        />
      </div>
      <v-btn
        class="button"
        color="primary"
        :disabled="isAnimating"
        @click="replay"
      >
        Replay <v-icon>mdi-play</v-icon>
      </v-btn>
      <div class="step-btn-container">
        <HistoryButtons
          id="step-buttons"
          class="mx-4"
          :current-frame="currentFrame"
          :length="currentSequence ? currentSequence.frames.length : 0"
          :disabled="stepButtonsAreDisabled"
          label="Animation"
          @HistoryEvents="changeCurrentFrame"
        />
      </div>
    </div>
  </div>
</template>

<script>

import Router from 'vue-router';
import InsertInput from './shared/InsertInput.vue';
import SearchInput from './shared/SearchInput.vue';
import DeleteInput from './shared/DeleteInput.vue';
import Visualizer from './shared/Visualizer.vue';
import HistoryButtons from './shared/HistoryButtons.vue';
import Sequence from '../assets/visualizer/frame';
import BTree, { BTreeNode } from '../assets/implementations/btree';

const router = new Router();

export default {
  name: 'TreeVisualizerHandler',
  components: {
    InsertInput,
    SearchInput,
    DeleteInput,
    Visualizer,
    HistoryButtons,
  },
  data() {
    return {
      /** @type {BTree} */
      tree: null,
      /** @type {Sequence[]} */
      sequencesList: [],
      insertionQueue: [],
      searchValueData: '',
      currentSequenceNumber: 0,
      currentFrame: 0,
      onlypop: true,
      isAnimating: false,
      title: '',
      Found: true,

    };
  },
  computed: {
    sequences() {
      return this.sequencesList;
    },
    currentSequence() {
      return this.sequencesList[this.currentSequenceNumber];
    },
    stepButtonsAreDisabled() {
      if (!this.currentSequence) {
        return {
          backButtonIsDisabled: true, forwardButtonIsDisabled: true,
        };
      }
      const backButtonIsDisabled = this.currentFrame <= 0;
      const forwardButtonIsDisabled = this.currentFrame >= this.currentSequence.frames.length - 1;
      return { backButtonIsDisabled, forwardButtonIsDisabled, isAnimating: this.isAnimating };
    },
    historyButtonsAreDisabled() {
      if (!this.sequencesList) {
        return { backButtonIsDisabled: true, forwardButtonIsDisabled: true };
      }
      const backButtonIsDisabled = this.currentSequenceNumber <= 0;
      const forwardButtonIsDisabled = this.currentSequenceNumber >= this.sequencesList.length - 1;
      return { backButtonIsDisabled, forwardButtonIsDisabled, isAnimating: this.isAnimating };
    },
  },
  mounted() {
    this.tree = new BTree(2);
    this.onlypop = false;
    this.title = 'B Tree';
    this.tree.root = new BTreeNode(true);
    this.tree.root.tree = this.tree;
  },
  methods: {
    goBack() {
      router.back();
    },
    insertInputEvent(event) {
      this.Found = true;

      event.split(',').forEach((value) => {
        if (value) {
          this.insertionQueue.push(value);
        }
      });
      this.dequeueInsert();
    },
    SetSearchValue(event) {
      this.searchValueData = event;
      this.Search();
    },
    dequeueInsert() {
      const newSequence = new Sequence();
      this.sequencesList.push(newSequence);
      this.currentSequenceNumber = this.sequencesList.length - 1;
      const sequence = this.tree.insert(this.insertionQueue[0]);
      this.insertionQueue.splice(0, 1);
      this.addSequenceAsync(sequence);
    },
    Search() {
      const newSequence = new Sequence();
      this.sequencesList.push(newSequence);
      this.currentSequenceNumber = this.sequencesList.length - 1;
      const { sequence, node } = this.tree.searchValue(this.searchValueData);
      if (!node) {
        this.Found = false;
      } else {
        this.Found = true;
      }
      console.log(node);
      console.log(this.notFound);
      this.addSequenceAsync(sequence);
    },
    deleteInputEvent(event) {
      if (event !== '') {
        this.Found = true;
        const newSequence = new Sequence();
        this.sequencesList.push(newSequence);
        this.currentSequenceNumber = this.sequencesList.length - 1;
        this.addSequenceAsync(this.tree.delete(event));
      }
    },
    replay() {
      const frames = [];
      this.currentSequence.frames.forEach((f) => frames.push(f));
      this.currentSequence.frames = [];
      this.addSequenceAsync({ frames });
    },
    addSequenceAsync(sequence) {
      this.isAnimating = true;
      this.currentSequence.addFrame(sequence.frames[0]);
      this.currentFrame = 0;
      for (let i = 1; i < sequence.frames.length; i += 1) {
        setTimeout(() => {
          this.currentSequence.addFrame(sequence.frames[i]);
          this.currentFrame += 1;
          this.sequencesList = Object.assign(this.sequencesList);
          if (i === sequence.frames.length - 1) {
            this.isAnimating = false;
            if (this.insertionQueue.length > 0) {
              this.dequeueInsert();
            }
          }
        }, i * 500);
      }
      this.isAnimating = false;
    },
    changeCurrentFrame(event) {
      this.currentFrame += event;
      if (this.currentFrame < 0) {
        this.currentFrame = 0;
      }
      if (this.currentFrame > this.sequencesList[this.currentSequenceNumber].frames.length - 1) {
        this.currentFrame = this.sequencesList[this.currentSequenceNumber].frames.length - 1;
      }
    },
    changeCurrentSequence(event) {
      this.currentSequenceNumber += event;
      if (this.currentSequenceNumber < 0) {
        this.currentSequenceNumber = 0;
      }
      if (this.currentSequenceNumber > this.sequencesList.length - 1) {
        this.currentSequenceNumber = this.sequencesList.length - 1;
      }
      this.currentFrame = this.currentSequence.frames.length - 1;
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .structies-button:hover {
    background-color:#3c48fa;
  }
  .back {
    position: absolute;
    left: 0;
    top: 0;
  }
  .buttons {
    place-content: space-around;

    flex: 0 0 auto;
  }
  .tree-container {
    width: 100%;
    height: 90vh;
    flex-direction: column;
  }
  .hidden{
    display: none;
  }
  .show{
    display: block;
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 4px;
  }
</style>
