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
      earth: {
        diameter_miles: 7917.5,
        diameter_meters: 12742000,
        container_diameter: 0,
      },
      size: {
        w: 0,
        h: 0,
      },
      svg: undefined,
    }
  },
  watch: {
    neo_list: function (v) {
      //console.log(this.size)
      this.setEarthRelativeDiameter()
      //console.log('erath', this.earth)
      this.drawSVG()
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
      })
      .catch( (response) => {
        console.error(response)
      })
    },
    setEarthRelativeDiameter: function () {
      let max = (this.size.w > this.size.h) ? this.size.h : this.size.w
      this.earth.container_diameter = this.earth.diameter_miles * (1 / 96)
      //console.log('earth, 1/96', this.earth.diameter_miles * (1/96))
    },
    setSVGSizes: function () {
      this.size.w = this.$refs.svg.offsetWidth
      this.size.h = this.$refs.svg.offsetHeight
    },
    avgEstDiameter: function (obj) {
      let aed = ((obj.estimated_diameter.meters.estimated_diameter_max
        + obj.estimated_diameter.meters.estimated_diameter_min) / 2)
      //console.log("aed", aed)
      let relative_diameter = aed / this.earth.diameter_meters
      //console.log('relative', relative_diameter)
      return aed * (1 / 96)
      //return (this.earth.container_diameter * relative_diameter)
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
      let radii = []
      this.neo_list.forEach( (v,i) => {
        arcs.push(this.drawNEO(0, this.avgEstDiameter(v)))
        radii.push(this.avgEstDiameter(v))
      })
      console.log(this.neo_list)
      //console.log(this.earth.container_diameter)

      // draw the sun
      // one 'au' is currently half the svg width
      let au = this.size.w / 2
      let sun = this.svg.append("g").attr("class", "sun")
        .selectAll().data(['sun'])
          .enter().append("circle")
          .attr("r", 100)
          .attr("cx", 3 *(this.size.w / 4))
          .attr("cy", this.size.h / 2)
          .attr("fill", "orange")
      // draw the earth
      let earth = this.svg.append("g")
          .attr("class", "earth")
        .selectAll().data([this.earth.container_diameter])
          .enter().append("circle")
          .attr("r", (d) => d/2)
          .attr("cx", 1 * (this.size.w / 4))
          .attr("cy", this.size.h / 2)
          .attr("fill", "green")

      // draw the NEOs 
      let circles = this.svg.append("g")
          .attr("class", "neos")
        .selectAll().data(radii)
      circles.enter().append("circle")
        .attr("r", (d) => d)
          // perihelion dist is the closest point to the sun in elip orbat.
        .attr("cx", (d,i) => this.neo_list[i].orbital_data.perihelion_distance * au) 
        .attr("cy", this.size.h / 2)
        .attr("fill", "blue")
        .attr("title", (d,i) => this.neo_list[i].name)
        .attr("stroke", (d,i) => {
          if (this.neo_list[i].is_potentially_hazardous_asteroid) {
            return "red"
          }
          return "yellow"
        })
        .attr("stroke-width", 2)

      
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
    this.setSVGSizes()
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
  width: 800px;
  height: 800px;
  margin: 0 auto;
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
