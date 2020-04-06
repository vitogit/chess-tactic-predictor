<template>
  <div class="blue merida">
    <div ref="board" class="cg-board-wrap"></div> </br>
    <a @click="takeBack" class="button">Take back</a>
    <a @click="analyzeMoves" class="button">Analyze Moves</a>
    <a @click="analyzeOpponentMoves" class="button">Analyze Opponent Moves</a>
    <a @click="openLichess" class="button">View on Lichess</a>
    
    <br><br><br>
    <div style="text-align:left">
      <ul>
        <li v-for="move in this.analyzedMoves">
          <b v-bind:class="{ 'bad':  move[1]< 0, 'good': move[1]>0 }">{{ move[0] }}:</b>  {{ move[1] }}
        </li>
        <li>(Good moves/bad moves)*100 = {{complexity}}</li>
        
      </ul>
    </div>  
    
  </div>
</template>

<script>
import Chess from 'chess.js'
import {Chessground} from 'chessground'
import {uniques} from './Util.js'

export default {
  name: 'chessboard',
  props: {
    ai_difficulty: {
      type: String,
      default: 'Normal',
    },  
    fen: {
      type: String,
      default: '',
    },
    free: {
      type: Boolean,
      default: false,
    },
    both: {
      type: Boolean,
      default: false,
    },
    onPromotion: {
      type: Function,
      default: () => 'q',
    },
  },
  data () {
    return {
      score: 0,
      ai_skill: '10',
      ai_depth: '8',
      scores:[],
      takingback: false,
      analyzedMoves: []
      
    }
  },
  watch: {
    ai_difficulty: function (ai_difficulty) {
      if (ai_difficulty == 'Easy') {
        this.ai_depth = '3'
        this.ai_skill = '0'
      } else if (ai_difficulty == 'Hard') {
        this.ai_depth = '12'
        this.ai_skill = '20'
      } else {
        this.ai_depth = '8'
        this.ai_skill = '10'
      }
    },    
    fen: function (newFen) {
      this.fen = newFen
      this.loadPosition()
    },
    both: function (newboth) {
      this.both = newboth
    }
  },
  methods: {
    opponentMoves () {
      let originalPGN = this.game.pgn()
      let tokens = this.game.fen().split(' ')
      tokens[1] = tokens[1] === 'w' ? 'b' : 'w'
      tokens = tokens.join(' ')
      let valid = this.game.load(tokens)
      if (valid) {
        let moves = this.game.moves({verbose: true})
        this.game.load_pgn(originalPGN)
        return moves
      } else {
        return []
      }
    },
    analyzeMoves: function () {
      this.analyzeOpponent = false
      this.analyzedMoves = []
      this.movesToAnalyze = this.game.moves({verbose: true})
      console.log("this.movesToAnalyze________",this.movesToAnalyze)
      this.nextAnalyzeMove()
    },  
    analyzeOpponentMoves: function () {
      var tokens = this.game.fen().split(' ');
      console.log("this.game.fen()________",this.game.fen())
      console.log("tokens________",tokens)
      tokens[1] = tokens[1] === 'w' ? 'b' : 'w'
      this.opponentGame = new Chess(tokens.join(' '))
      this.analyzeOpponent = true
      this.analyzedMoves = []
      this.movesToAnalyze = this.opponentGame.moves({verbose: true})
      this.nextAnalyzeMove()
    },
    nextAnalyzeMove: function () {
      if (this.movesToAnalyze == null || this.movesToAnalyze.length == 0) {
        this.analyzedMoves = this.analyzedMoves.sort(function(a, b) {
            return b[1] - a[1];
        });
        this.paintThreats()
      } else {
        let m = this.movesToAnalyze.pop()
        let movesUci = null
        if (this.analyzeOpponent) {
          this.opponentGame.move(m)
          this.uci('position fen  ' + this.opponentGame.fen(), 'analyzer')
        } else {
          this.game.move(m)
          this.uci('position fen  ' + this.game.fen(), 'analyzer')
        }
        // this.uci('ucinewgame', 'analyzer')
        // this.uci('position startpos moves ' + movesUci, 'analyzer')
        this.uci("go depth 10",'analyzer')
      }

    },  
    movesToUci(game) {
      let moves = ''
      let history = game.history({verbose:true})
      for(let i = 0; i < history.length; ++i) {
        let move = history[i]
        moves += ' ' + move.from + move.to + (move.promotion ? move.promotion : '')
      }
      return moves
    },    
    takeBack() {
      this.takingback = true
      this.game.undo()
      this.game.undo()

      this.scores.pop()
      this.sendInfo()
    },
    uci(cmd, engine = 'ai') {
      if (engine == 'ai') {
        this.ai.postMessage(cmd)
      } else if (engine == 'brain') {
        this.brain.postMessage(cmd)
      } else {
        this.analyzer.postMessage(cmd)
      }
    },
    possibleMoves () {
      const dests = {}
      this.game.SQUARES.forEach(s => {
        const ms = this.game.moves({square: s, verbose: true})
        if (ms.length) dests[s] = ms.map(m => m.to)
      })
      return dests
    },
    toColor () {
      return (this.game.turn() === 'w') ? 'white' : 'black'
    },
    paintThreats () {
      let temp =  [].concat(this.analyzedMoves);
      // let best3Moves = temp.splice(0,3)
      let bestMoveScore = temp[0][1]
      let threats = []
      temp.forEach(function (move) {
        if (bestMoveScore - parseInt(move[1]) >= 100   ) {
          threats.push({orig: move[0].from, dest: move[0].to, brush: 'red'})
        } else if (bestMoveScore - parseInt(move[1]) <= 30) {
          console.log("move[0]________",move[0])
          console.log("move[3]________",move[3])
          console.log("move[0]==move[3]________",move[0]==move[3])
          if (move[0].to == move[3].to && move[0].from == move[3].from) {
            threats.push({orig: move[0].from, dest: move[0].to, brush: 'green'})
          } else {
            threats.push({orig: move[0].from, dest: move[0].to, brush: 'green'})
            threats.push({orig: move[3].from, dest: move[3].to, brush: 'yellow'})
          }

        }
      })

      // best3Moves.forEach(function (move) {
      //   threats.push({orig: move[0].from, dest: move[0].to, brush: 'green'})
      // })
      console.log("threats________",threats)
      this.board.setShapes(threats)
    },
    calculatePromotions () {
      let moves = this.game.moves({verbose: true})
      this.promotions = []
      for (let move of moves) {
        if (move.promotion) {
          this.promotions.push(move)
        }
      }
    },
    isPromotion   (orig, dest) {
      let filteredPromotions = this.promotions.filter(move => move.from === orig && move.to === dest)
      return filteredPromotions.length > 0 // The current movement is a promotion
    },
    userPlay() {
      return (orig, dest) => {
        if (this.isPromotion(orig, dest)) {
          this.promoteTo = this.onPromotion()
        }
        this.game.move({from: orig, to: dest, promotion: this.promoteTo}) // promote to queen for simplicity
        this.board.set({
          fen: this.game.fen(),
          animation: {
            duration: 500
          },
        })
        this.calculatePromotions()
        this.afterMove()
        // this.analyzeMoves()
      };
    },
    afterMove () {
      if (this.isFinished() ) {
        return
      }
      if (this.both) { //The player wants to play against himself
        this.board.set({
          turnColor: this.toColor(),
          movable: {
            color: this.toColor(),
            dests: this.possibleMoves(),
          }
        });
      } else {
        //Ai next move
        this.uci("position fen " + this.game.fen())
        this.uci(`go depth ${this.ai_depth}`)
      }
      this.sendInfo()
    },
    sendInfo() {
      let threats = {}
      threats['history'] = this.game.history()
      threats['fen'] = this.game.fen()
      threats['pgn'] = this.game.pgn()
      this.$emit('onMove', threats)
    },
    isFinished() {
      if (this.game.in_checkmate()) {
        if (this.toColor() == 'black') {
          this.$emit('onEnd', "YOU WIN")
        } else {
          this.$emit('onEnd', "YOU LOSE")
        }
        return true
      } else if (this.game.in_draw() ) {
        this.$emit('onEnd', "DRAW")
        return true
      } else if (this.game.in_stalemate() ) {
        this.$emit('onEnd', "STALEMATE")
        return true
      }
      return false

    },
    loadPosition () { // set a default value for the configuration object itself to allow call to loadPosition()
      this.game.load(this.fen)
      this.board = Chessground(this.$refs.board, {
        fen: this.game.fen(),
        turnColor: this.toColor(),
        movable: {
          color: this.toColor(),
          free: this.free,
          dests: this.possibleMoves(),
        },
      })
      this.board.set({
        movable: { events: { after: this.userPlay()} },
      })
      // this.afterMove()
    },
    openLichess() {
      let url = 'http://lichess.org/analysis/'+this.game.fen()
       window.open(url, "_blank"); 
    },
    startAi() {
      this.ai = STOCKFISH();
      this.uci('uci')
      this.uci("ucinewgame")
      this.uci("isready")
      let self = this
      this.ai.onmessage = function(event) {
        if (self.toColor() == 'white') {
          return
        }

        var line;
        if (event && typeof event === "object") {
            line = event.data;
        } else {
            line = event;
        }
        /// Ignore some output.
        if (line === "uciok" || line === "readyok" || line.substr(0, 11) === "option name") {
            return;
        }
        var match = line.match(/^bestmove ([a-h][1-8])([a-h][1-8])([qrbn])? .* (cp (-?\d+)|mate (-?\d+))/)
        /// Did the AI move?
        if(match) {
            let result = match[5]
            if (match[4].includes('mate')) {
              if (parseInt(match[6]) > 0) {
                result = 10000
              } else {
                result = -10000
              }
            }
            self.game.move({from: match[1], to: match[2], promotion: match[3]});

            self.calculatePromotions()

            self.$emit('onMove', {pgn: self.game.pgn()})
            self.board.set({
              fen: self.game.fen(),
              animation: {
                duration: 500
              },
              turnColor: self.toColor(),
              movable: {
                color: self.toColor(),
                dests: self.possibleMoves(),
              }
            })
            if (self.isFinished() ) {
              return
            }
        }
      }
    },

    startAnalyzer() {
      this.analyzer = STOCKFISH();
      this.uci('uci','analyzer')
      this.uci("ucinewgame",'analyzer')
      this.uci("isready",'analyzer')
      let self = this

      this.analyzer.onmessage = function(event) {
        var line;
        if (event && typeof event === "object") {
            line = event.data;
        } else {
            line = event;
        }
        console.log("event________",event)
        /// Ignore some output.
        if ( line == undefined || line.includes("info") || line === "uciok" || line === "readyok" || line.substr(0, 11) === "option name") {
            return;
        }
        console.log("line________",line)
        var match = line.match(/^bestmove ([a-h][1-8])([a-h][1-8])([qrbn])? .* ponder (.*) ponderSan .* (cp (-?\d+)|mate (-?\d+))/)
        console.log("match________",match)
        let lastMove = null

        if(match) {
          if (self.analyzeOpponent) {
            lastMove = self.opponentGame.undo()
          } else {
            lastMove = self.game.undo()
          }
          let bestReply = { from: match[1], to: match[2] }
          let nextMove = { from: match[4].slice(0,2), to: match[4].slice(2,4) }
          console.log("self.toColor()________",self.toColor())
          let score = parseInt(match[6])
          if (match[3] && match[3].includes("mate")) {
            score = match[3].split(" ")[1]
            score = 10000 * score
          }
          score = (self.toColor() == 'white' ? parseInt(score*-1) : parseInt(score))

          self.analyzedMoves.push([lastMove, score, bestReply, nextMove])
          self.nextAnalyzeMove()
        } else if(line.includes("mate")) {
          let match = line.match(/bestmove (.*)/)
          let score = 10000
          if (self.analyzeOpponent) {
            lastMove = self.opponentGame.undo()
          } else {
            lastMove = self.game.undo()
          }          
          let bestReply = { from: match[1].slice(0,2), to: match[1].slice(2,4) }
          let nextMove = { from: "", to: "" }
          self.analyzedMoves.push([lastMove, score, bestReply, nextMove])
          self.nextAnalyzeMove()
        
        }
      };
    }
  },
  mounted () {
    this.loadPosition()
    this.startAi()
    this.startAnalyzer()
  },
  created () {
    this.game = new Chess()
    this.opponentGame = new Chess()
    this.analyzeOpponent = false
    this.board = null
    this.promotions = []
    this.promoteTo = 'q'
    this.movesToAnalyze = []
    this.complexity = 0
  },
}
</script>
