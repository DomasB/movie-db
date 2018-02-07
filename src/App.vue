<template>
  <div id="app">
    <input v-model="text" @keyup.enter="search(text)" type="text">
    <input v-model="activation"  type="text">
    <button @click="search(text)">Search</button>
    <button @click="transformData()">Transform</button>
    <button @click="train()">Train</button>
    <button @click="save()">Save</button>
    <button @click="load()">Load</button>
    <ul>
      <li v-for="(movie, index) in movies">
        {{movie.Title}} - {{movie.imdbRating | toInt}} - {{movie.Year}} - {{movie.imdbVotes}} - {{movie.Runtime}}
        <input v-model="movie.myScore" type="range" min="0" max="10"></input>
        {{movie.myScore}}
        <button @click="deleteMovie(index)">X</button>
        <button @click="run(index)">Run</button>
      </li>
    </ul>
  </div>
</template>

<script>
const brain = require('brain.js');
var net;
export default {
  name: 'app',
  data () {
    return {
      movies:[],
      data:[],
      activation: 'sigmoid',
      text:""
    }
  },
  methods: {
    search(text) {
      var url = 'http://www.omdbapi.com/?t='+text+'&apikey=570a31f7';
      var _this = this;
      this.text = "";
      fetch(url, {
        method:'GET'
      }).then(res => res.json())
        .catch(error => console.error(error))
        .then(response => {
          response.myScore = 0;
          console.log(response);
          _this.movies.unshift(response);
        });
    },
    deleteMovie(id) {
     this.movies.splice(id, 1); 
    },
    transformData() {
      var findMax = function(property, arr) {
        return Math.max(...arr.map(o => o[property]));
      }
      var findMin = function(property, arr) {
        return Math.min(...arr.map(o => o[property]));
      } 
      var normalize = function(property, arr) {
        var max = findMax(property, arr)
        var min = findMin(property, arr)
        arr.map(o => {
          o[property] = (o[property] - min)/(max-min)
        })
      }
      var mappedData = this.movies.map(movie => {
        var imdbRating = parseFloat(movie.imdbRating);
        var imdbVotes = parseInt(movie.imdbVotes.replace(",",""));
        var Runtime = parseInt(movie.Runtime.replace(" mins",""));
        return {imdbRating, imdbVotes, Runtime, Year: movie.Year, Metascore: movie.Metascore, myScore: movie.myScore};
      })
      normalize('imdbRating', mappedData);
      normalize('imdbVotes', mappedData);
      normalize('Year', mappedData);
      normalize('Runtime', mappedData);
      normalize('Metascore', mappedData);
      this.data = mappedData.map(o =>{
        return {input: { imdb: o.imdbRating, year: o.Year, votes: o.imdbVotes, time: o.Runtime, mscore: o.Metascore}, output: { score: o.myScore/10 }};
      });
      console.log(this.data);
    },
    train() {
      net = new brain.NeuralNetwork({activation: this.activation, hiddenLayers:[5,2]});
      net.train(this.data, { iterations: 40000, log:true });
    },
    run(index) {
      var input = this.data[index].input
      this.movies[index].myScore = net.run(input).score*10;
    },
    save() {
      window.localStorage.setItem('movies', JSON.stringify(this.movies));
    },
    load() {
      this.movies = JSON.parse(window.localStorage.getItem('movies'));
    }

  },
  filters: {
    toInt(value, divider = 10) {
      return parseFloat(value/divider)
    }
  }
}
</script>

<style>
</style>
