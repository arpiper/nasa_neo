<template>
  <div id="nasa-near-earth-objects">
    <div class="near-earth-objects">
      <span v-for="neo, index in neo_list" class="neo" :class="{'selected': (index === 0)}">
        <span @click="drawInfoBox([neo], $event)">{{ neo.name }}</span>
      </span>
    </div>
    <div ref="svg" class="svg-container" id="neo-svg">
    </div>
    <div class="controls">
      <span>Scaling: <input v-model="scale_factor" type="number"></span>
      <span>
        <button @click="stopTimer(earth.timer)">
          Pause Earth
        </button>
      </span>
      <span>
        <button @click="stopTimer(timer)">
          Pause NEO's
        </button>
      </span>
      <span>
        <button @click="earth.timer = setEllipticalTimer(earth.orbit)">
          Start Earth
        </button>
      </span>
      <span>
        <button @click="timer = setEllipticalTimer()">
          Start NEO's
        </button>
      </span>
    </div>
    <div class="fine-print">Objects not to scale</div>
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
        name: "Earth",
        orbital_data: {
          eccentricity: 0.0167,
          semi_major_axis: 1.00000011,
          orbital_period: 365,
        },
        radius: 15,
        colors: {
          fill: "blue",
          stroke: "blue",
          orbit: "blue",
        },
        orbit: undefined,
        timer: undefined,
      },
      size: {
        w: 0,
        h: 0,
      },
      au: 1,
      scale_factor: 4,
      svg: undefined,
      orbits: undefined,
      timer: undefined,
    }
  },
  watch: {
    neo_list: function (v) {
      this.setEarthRelativeDiameter()
      this.drawSVG()
    },
    scale_factor: function (v) {
      this.au = this.size.w / v
      d3.select("#neo-svg svg").remove()
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
      this.earth.radius = this.earth.diameter_miles * (1 / 3840)
    },
    setSVGSizes: function () {
      this.size.w = this.$refs.svg.offsetWidth
      this.size.h = this.$refs.svg.offsetHeight
      this.au = this.size.w / this.scale_factor
    },
    avgEstDiameter: function (obj) {
      let aed = ((obj.estimated_diameter.meters.estimated_diameter_max
        + obj.estimated_diameter.meters.estimated_diameter_min) / 2)
      return aed * (1 / 96)
    },
    drawSVG: function () {
      this.svg = d3.select("#neo-svg").append("svg")
        .attr("height", "100%")
        .attr("width", "100%")
        .attr("x", 0)
        .attr("y", 0)
        .attr("viewBox", `0 0 ${this.$refs.svg.offsetWidth} ${this.$refs.svg.offsetHeight}`)
        .attr("preserveAspectRatio", "none")
      let colors = d3.schemeCategory20
      this.neo_list.forEach( (v,i) => {
        this.neo_list[i].radius = this.avgEstDiameter(v)
        this.neo_list[i].colors = {
          fill: "gray",
          stroke: (v.is_potentially_hazardous_asteroid) ? "red" : "yellow",
          orbit: colors[i]
        }
      })
      // draw the sun
      let sun = this.svg.append("g").attr("class", "sun")
          .attr("class", "sun")
        .selectAll().data(['sun'])
          .enter().append("circle")
          .attr("r", 40)
          .attr("cx", 3 * (this.size.w / 4))
          .attr("cy", this.size.h / 2)
          .attr("fill", "orange")
      // draw the earth
      this.drawEllipticPath([this.earth], "earth")
      this.earth.orbit = this.drawMovingOrbit([this.earth], "earth-orbit")
      this.earth.timer = this.setEllipticalTimer(this.earth.orbit)
      // draw neo orbital path ellipse
      let ellipses = this.drawEllipticPath(this.neo_list)
      // draw neo moving along their orbital path
      this.orbits = this.drawMovingOrbit(this.neo_list)
      // Start orbital motion
      this.timer = this.setEllipticalTimer(this.orbits);
      // create info box
      this.drawInfoBox([this.neo_list[0]])
    },
    drawEllipticPath: function (object_list, classname = "ellipses") {
      return this.svg.append("g")
          .attr("class", classname)
        .selectAll().data(object_list)
          .enter().append("ellipse")
          .attr("stroke", (d,i) => d.colors.orbit)
          .attr("stroke-width", 1)
          .attr("fill", "transparent")
          .attr("rx", (d) => d.orbital_data.semi_major_axis * this.au)
          .attr("ry", (d) => {
              // calculate the semi-minor-axis of the ellipse -> a*sqrt(1 - e^2)
              let smi = (d.orbital_data.semi_major_axis) * Math.sqrt(1 - Math.pow(d.orbital_data.eccentricity,2))
              return smi * this.au
          })
          .attr("cx", (d) => (3 * (this.size.w / 4)) - (d.orbital_data.eccentricity * d.orbital_data.semi_major_axis * this.au))
          .attr("cy", this.size.h / 2)
    },
    drawMovingOrbit: function (object_list, classname = "moving") {
      // neo container element
      let moving = this.svg.append("g")
          .attr("class", classname)
          .attr("transform", `translate(${3*(this.size.w / 4)}, ${this.size.h / 2})`)
        .selectAll().data(object_list)
          .enter().append("g")
      // neo circle representation
      this.svg.select(`g.${classname}`).selectAll("g")
        .append("circle")
        .attr("r", (d) => d.radius * 5)
        .attr("fill", (d) => d.colors.fill)
        .attr("stroke", (d) => d.colors.stroke)
        .attr("stroke-width", "1")
      // neo name tag
      moving.append("text")
        .attr("x", (d) => d.radius * 4)
        .attr("dy", (d) => d.radius * 4)
        .attr("font-size", "12")
        .text((d,i) => {
          return `${d.name}`
        })
      // neo orbital movement function
      moving.datum((d,i) => {
        return (j) => {
          // angle * scale factor
          let a = (j / d.orbital_data.orbital_period) * Math.PI * (1 / 16) 
          let r = (d.orbital_data.semi_major_axis 
            * (1 - Math.pow(d.orbital_data.eccentricity,2))) 
            / (1 + (d.orbital_data.eccentricity * Math.cos(a)))
          let cx = (r * this.au) * Math.cos(a)
          let cy = (r * this.au) * Math.sin(a)
          return {cx: cx, cy: cy}
        }
      })
      return moving
    },
    drawInfoBox: function (dataObject, evt) {
      if (evt) {
        this.removeSelected()
        evt.target.parentNode.className += " selected"
      }
      if (!this.svg.select("g.info-box").empty()) {
        this.svg.select("g.info-box").remove()
      }
      let box = this.svg.append("g")
          .attr("class", "info-box").selectAll()
          .data(dataObject)
        .enter().append("foreignObject")
          .attr("x", 20)
          .attr("y", this.size.h * 2 / 3)
          .attr("width", 150)
          .attr("height", 150)
          .attr("class", "neo-info-box")
        .append("xhtml:body")
          .style("margin", "5px")
      // NEO name
      box.append("div")
        .text((d) => d.name)
      // Estimated diameter range in miles
      box.append("div").text((d) => {
          let min = d.estimated_diameter.miles.estimated_diameter_min.toPrecision(3)
          let max = d.estimated_diameter.miles.estimated_diameter_max.toPrecision(3)
          return `Est. Diameter(miles): ${min} - ${max}`
        })
      // link to NASA Jet Propulsion Lab
      box.append("div").append("a")
          .attr("href", (d) => d.nasa_jpl_url)
          .attr("target", "_blank")
          .attr("rel", "noopener")
          .text("NASA JPL")
      // Is potentially hazardous
      box.append("div").text((d) => {
        let b = (d.is_potentially_hazardous_asteroid) ? "Y" : "N"
        return `Is Potentially Hazardous?: ${b}`
      })
      box.append("div").text(
        (d) => `Relative Velocity: ${d.close_approach_data[0].relative_velocity.miles_per_hour} mph`
      )
      box.append("div")
        .style("color", (d) => d.colors.orbit)
        .text(
        (d) => `Orbit Line Color: ${d.colors.orbit}`
      )
    },
    setEllipticalTimer: function (objects = this.orbits) {
      let timer = d3.timer((i) => {
        objects.attr("transform", (d) => {
          let c = d(i)
          return `translate(${c.cx}, ${c.cy})`
        })
      })
      return timer
    },
    stopTimer: function (timer) {
      timer.stop()
    },
    removeSelected: function () {
      let s = document.querySelector(".selected")
      let i = s.className.split(" ")
      i.splice(i.indexOf("selected"),1)
      s.className = i.join(" ")
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
#nasa-near-earth-objects {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  width: 80%;
  display: flex;
  justify-content: center;
  margin: 0 auto;
  flex-wrap: wrap;
}
#neo-svg {
  border: 1px solid #444;
  width: 100%;
  height: 500px;
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
.near-earth-objects {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
}
.neo {
  padding: 3px;
  margin: 2px;
  background-color: #42b983;
  color: #f3f3f3;
  border: 3px solid #42b983;
}
.neo:hover,
.neo.selected {
  color: #42b983;
  background-color: #f3f3f3;
}
.neo:hover {
  cursor: pointer;
}
.neo-info-box {
  background-color: #ddd;
  font-size: 10px;
  text-align: left;
}
.neo-info-box div {
  padding: 2px 0;
}
.fine-print { 
  font-size: 8pt;
  padding: 5px;
}
</style>
