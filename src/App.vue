<template>
  <div id="app">
    <section class="hero is-fullheight">
      <div class="hero-head">
        <nav class="navbar">
          <div class="container">
            <div>
              <a href="/">
                <p class="title">Chess Tactic Predictor (beta)</p>
                <p class="subtitle">Show the current and future threats. <br> <br><a @click.prevent="moreInfo=true"> Learn more about this project</a> or
                <a href="http://twitter.com/share?text=Show chess current and future threats.;url=http://vitomd.com/chess-tactical-opportunity&amp;hashtags=chess&amp;via=vitomd"
                onclick="window.open(this.href, 'twitter-share', 'width=550,height=235');return false;">
                Share it on Twitter
                </a>
              </p>
              </a>
            </div>
          </div>
        </nav>
      </div>
      <div class="hero-body" style="padding-bottom:0; padding-top:10px">
        <div class="container has-text-centered">
          <div class="columns" style="height:400px">
            <div class="column is-5">
              <chessboard @onMove="showInfo" 
                          :key="chessboardId" 
                          @onEnd="onEnd" 
                          @onScore="showScore"
                          :ai_difficulty="ai_difficulty"
                          :both="both"
                          />
            </div>
            <div class="column is-7">
              <b-field position="is-centered">
                  <b-radio-button v-model="ai_difficulty"
                      native-value="Easy">
                      <span>Easy</span>
                  </b-radio-button>

                  <b-radio-button v-model="ai_difficulty"
                      native-value="Normal">
                      <span>Normal</span>
                  </b-radio-button>
                  
                  <b-radio-button v-model="ai_difficulty"
                      native-value="Hard">
                      <span>Hard</span>
                  </b-radio-button>
              </b-field>
              <br>
              <div class="field">
                  <b-switch v-model="both">
                      Play both sides
                  </b-switch>
              </div>

              <div v-if="moreInfo" style="text-align:left" class="help">
                <p>
                  It uses Stockfish to play and analize the tactical opportunities. You can play white or black and also freemode where you can select a position to play against the computer.
                  Useful when trying to play the middlegame of a opening.
                </p>
                <p>
                  Select the difficult mode. Easy(stockfish: 0.1s, 1 skill), Normal(stockfish:0.5s, 10 skill), Hard(stockfish:1s, 20 skill).
                </p>
                <p>
                  The AI will make mistakes if playing in easy or normal mode, so sometimes you will get a tactical opportunity. A message will show up and tell you how much 
                  centipawns you can gain if you play the best move.
                </p>
                <p>
                  You can click to Analyze the best available moves in the position.
                </p>
                <br>
                <p>
                  I'm <a href="http://vitomd.com">@vitomd</a> a Software Engineer and chess lover. I made some other chess related projects:<br>
                  <a href="http://vitomd.com/vue-chess-guardian/">Chess Guardian</a>: Answer Chess positional questions from your own games. <br>
                  <a href="https://github.com/vitogit/vue-chessboard">Vue Chessboard Component</a>: Load positions, create positions and see threats.
                  <a href="http://handandchess.com">Hand and Brain Chess</a>: A fun chess variation to play in your device agains the computer.
                </p>
                <a @click.prevent="moreInfo=false" class="button">Keep playing</a>

              </div>
              <h1 v-if="!moreInfo" class="title">
               <div v-show="endStyle" class ="shout"  v-bind:class="[endStyle]">{{endText}}</div>
               <div >
                 <a @click.prevent="newGame" class="button is-large">New Game</a>
               </div>
               <div style="font-size:0.4em; padding:10px">
                 {{this.positionInfo}}
               </div>
              </h1>
            </div>
          </div>
        </div>
      </div>

      <div  class="hero-foot">
        <div class="container">
          <div v-if="!moreInfo" class="has-text-centered">
            Made by <a href="http://vitomd.com">@vitomd</a> with <a href="https://vuejs.org/">Vue.js</a> | Check all my <a href="http://vitomd.com/blog/projects/">chess related projects</a>
          </div>
        </div>
      </div>
    </section>
  </div>

</template>

<style>
  #app {
    margin: 1vh;
  }
  .hero.is-fullheight {
    min-height: 95vh;
  }
  .cg-board-wrap {
    margin: 0 auto;
  }
</style>

<script>
import chessboard from './components/chessboard'
import './components/style/theme.css'

export default {
  name: 'app',
  components: {
    chessboard
  },
  data () {
    return {
      currentFen: '',
      moveMessage: "Play the first move",
      firstMove: true,
      moveStyle: "",
      evalStyle: "",
      endStyle: "",
      endText: "",
      moreInfo: false,
      positionInfo: null,
      tacticalOpportunity: '',
      ai_difficulty: 'Normal',
      both: false,
      movesCount: 0
    }
  },
  methods: {

    onEnd(end) {
      this.endText = end
      if (end == "YOU WIN") {
        this.endStyle = "perfect"
      } else if (end == "YOU LOSE") {
        this.endStyle = "worst"
      } else {
        this.endStyle = "draw"
      }
    },
    showScore(scoreDiff, score) {
      if (this.firstMove) {
        this.firstMove = false
        return
      }

      
      this.movesCount = this.positionInfo.length
      
    },
    loadFen(fen) {
      this.currentFen = fen
    },
    newGame() {
      Object.assign(this.$data, this.$options.data());
      this.chessboardId++
    },
    showInfo(data) {
      this.positionInfo = data.pgn
    },
  },
  created() {
    this.points = 0
    this.chessboardId = 0
  }
}
</script>

<style>
  .perfect {
    background-color: #8bc34a;
    color: black;
    font-size: 1.5em;
  }

  .good {
    color: #8bc34a;
  }

  .bad {
    color: #f44336;
  }

  .worst {
    background-color: #f44336;
    color:black;
    font-size: 1.5em;
  }

  .shout {
    padding: 10px;
    border-radius: 10px;
    margin: 20px;
  }

  .bright {
    color: black;
    background-color: yellow;
  }
  .help p {
    margin: 5px 0;
    font-size: 1.4em;
  }
</style>


<style>
  #app {
    margin: 1vh;
  }
  .hero.is-fullheight {
    min-height: 95vh;
  }
  .cg-board-wrap {
    margin: 0 auto;
  }
</style>
