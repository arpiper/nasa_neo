<template>
  <div id="app">
    <div class="near-earch-objects">
      {{ neo_data.element_count }}
    </div>
    <div v-for="object in neo_list" >
      {{ object.name }}
    </div>
    <div ref="svg" class="" id="neo-svg">
    </div>
  </div>
</template>

<script>
import nasa from "../nasa.json"
let axios = require("axios")
let d3 = require("d3")

export default {
  name: 'app',
  data () {
    return {
      api_key: nasa.api_key,
      neo_url: `${nasa.url}/neo/rest/v1`,
      neo_data: {},
      neo_list: undefined,
      svg: undefined,
    }
  },
  methods: {
    getNEOFeedToday: function () {
      let vm = this
      axios.get(`${this.neo_url}/feed/today`, {
        params: {
          detailed: true,
          api_key: this.api_key,
        }
      }).then( (response) => {
        vm.neo_data = response.data
        let d = new Date().toISOString()
        let t = response.data.near_earth_objects
        vm.neo_list = t[Object.keys(t)[0]]
        //vm.drawSVG()
      })
      .catch( (response) => {
        console.error(response)
      })
    },
    avgEstDiameter: function (obj) {
      return (obj.estimated_diameter.meters.estimated_diameter_max
        + obj.estimated_diameter.meters.estimated_diameter_min) / 2
    },
    drawSVG: function () {
      this.svg = d3.select("#neo-svg").append("svg")
        .attr("height", "100%")
        .attr("width", "100%")
        .attr("x", 0)
        .attr("y", 0)
        .attr("viewBox", `0 0 ${this.$refs.svg.offsetWidth} ${this.$refs.svg.offsetHeight}`)
        .attr("preserveAspectRatio", "none")
      let arcs = []
      console.log("neo", this.neo_list)
      this.neo_list.forEach( function (v,i) {
        arcs.push(this.drawNEO(0, this.avgEstDiameter(v)))
      })
      console.log("arcs", arcs)

      this.svg.append("g")
          .attr("class", "arcs")
        .selectAll().data(arcs)
          .enter().append("path")
          .attr("stroke", "black")
          .attr("stroke-width", 5)
          .attr("data-valueset", (d) => {console.log('h',d)})
          .attr("d", (d) => d)
    },
    drawNEO: function (ir, or) {
      let a = d3.arc({
        innerRadius: ir,
        outerRadius: or,
        startAngle: 0,
        endAngle: (2 * Math.PI),
      })
      return a
    },
  },
  created () {
    this.getNEOFeedToday()
  },
  mounted () {
    this.drawSVG()
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

#neo-svg {
  width: 600px;
  height: 400px;
  margin: 0 auto;
  border: 1px solid black;
}

h1, h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
