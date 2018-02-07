<template>
  <div id="app">
    <input v-model="text" @keyup.enter="search(text)" type="text">
    <input v-model="activation"  type="text">
    <button @click="search(text)">Search</button>
    <button @click="train()">Train</button>
    <button @click="save()">Save</button>
    <button @click="load()">Load</button>
    <button @click="sortMovies()">Sort</button>
    <ul>
      <li v-for="(movie, index) in movies">
        {{movie.Title}} - {{movie.imdbRating | toInt}} - {{movie.Year}} - {{movie.imdbVotes}} - {{movie.Runtime}} - {{movie.BoxOffice}}
        <input v-model="movie.myScore" type="range" min="0" max="10"></input>
        {{movie.myScore}}
        <button @click="deleteMovie(index)">X</button>
        <button @click="run(index)" :disabled="!trained">Run</button>
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
      text:"",
      trained: false
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
      var genres = [];
      var genreArray = [];
      this.movies.map(movie => {
        movie.Genre.split(', ').map(genre => {
          if (!genres.includes(genre)) genres.push(genre);
        })
      })
      var mappedData = this.movies.map(movie => {
        var genreObject = {}
        var imdbRating = parseFloat(movie.imdbRating);
        imdbRating = isNaN(imdbRating) ? 0 : imdbRating;
        var imdbVotes = parseInt(movie.imdbVotes.replace(",",""));
        imdbVotes = isNaN(imdbVotes) ? 0 : imdbVotes;
        var Metascore = isNaN(movie.Metascore) ? 0 : movie.Metascore;
        var Year = isNaN(movie.Year) ? 0 : movie.Year;
        var Runtime = parseInt(movie.Runtime.replace(" mins",""));
        var Runtime = isNaN(Runtime) ? 0 : Runtime;
        var BoxOffice = (!movie.BoxOffice || movie.BoxOffice == "N/A") ? 0 : parseInt(movie.BoxOffice.replace(/\$|,/ig, ""));
        var movieGenres = movie.Genre.split(', ');
        genres.map(genre => {
          genreObject[genre] = movieGenres.includes(genre) ? 1 : 0;
        })
        genreArray.push(genreObject);
        return {Runtime, BoxOffice, imdbRating, imdbVotes, Year, Metascore, myScore: movie.myScore};

      })

      normalize('imdbRating', mappedData);
      normalize('imdbVotes', mappedData);
      normalize('Year', mappedData);
      normalize('Runtime', mappedData);
      normalize('Metascore', mappedData);
      normalize('BoxOffice', mappedData);

      this.data = mappedData.map((o,i) => {
        return {input: Object.assign(genreArray[i], { imdb: o.imdbRating, year: o.Year, votes: o.imdbVotes, time: o.Runtime, box: o.BoxOffice, mscore: o.Metascore}), output: { score: o.myScore/10 }};
      });
console.log(this.data)
    },
    train() {
      this.transformData()
      net = new brain.NeuralNetwork({activation: this.activation});
      net.train(this.data, { iterations: 20000, log: true, logPeriod: 9999,  errorThresh: 0.002 });
      this.trained = true;
    },
    run(index) {
      this.transformData()
      var input = this.data[index].input
      this.movies[index].myScore = net.run(input).score*10;
    },
    save() {
      window.localStorage.setItem('movies', JSON.stringify(this.movies));
    },
    load() {
      this.movies = JSON.parse(window.localStorage.getItem('movies'));
    },
    sortMovies() {
      this.movies = this.movies.sort((a,b) => {
        return parseInt(a.myScore) < parseInt(b.myScore) ? 1 : -1
      })
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
