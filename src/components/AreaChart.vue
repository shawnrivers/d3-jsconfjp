<template>
  <div class="areachart">
    <svg :width="width" :height="height">
      <!-- MOVIE ARCS -->
      <path :key="d.id" v-for="d in arcs" :d="d.path" :fill="d.fill" stroke="#fff" />
      <!-- AXES -->
      <g ref="xAxis" :transform="`translate(0, ${y0})`" />
      <g ref="yAxis" :transform="`translate(${margin.left}, 0)`" />
    </svg>
  </div>
</template>

<script>
import _ from "lodash";
import * as d3 from "d3";
import { TweenLite } from "gsap";

const width = 1000;
const height = 300;
const margin = { top: 0, right: 0, bottom: 20, left: 60 };

export default {
  name: "areachart",
  props: ["movies"],
  data() {
    return {
      width,
      height,
      margin,
      y0: 0,
      arcs: []
    };
  },
  watch: {
    movies() {
      this.calculateScale();
      this.calculateData();
    }
  },
  methods: {
    calculateScale() {
      console.log(this.movies);

      if (this.movies.length === 0) {
        return;
      }

      // x scale with dates
      const [minDate, maxDate] = d3.extent(this.movies, d => d.date);
      this.xScale = d3
        .scaleTime()
        .domain([
          d3.timeMonth.offset(minDate, -2),
          d3.timeMonth.offset(maxDate, 2)
        ])
        .range([margin.left, width - margin.right]);

      // y scale with boxOffice - medianBox
      // Use the median as the baseline
      this.medianBox = d3.median(this.movies, d => d.boxOffice);
      const minMaxBox = d3.extent(
        this.movies,
        d => d.boxOffice - this.medianBox
      );

      this.yScale = d3
        .scaleLinear()
        .domain(minMaxBox)
        .range([height - margin.bottom, margin.top]);

      this.y0 = this.yScale(0);

      console.log(this.y0);

      // Update area generater
      const minMaxScore = d3.extent(this.movies, d => d.score);
      this.colorScale = d3.scaleSequential(d3.interpolateViridis).domain(minMaxScore).nice();

      this.area = d3.area().curve(d3.curveCatmullRom).y0(this.y0);
    },
    calculateData() {
      if (!this.movies.length) {
        return;
      }

      this.arcs = this.movies.map(d => ({
        id: d.id,
        path: this.area([
          [this.xScale(d3.timeMonth.offset(d.date, -2)), this.y0],
          [this.xScale(d.date), this.yScale(d.boxOffice - this.medianBox)],
          [this.xScale(d3.timeMonth.offset(d.date, 2)), this.y0]
        ]),
        fill: this.colorScale(d.score)
      }));

      console.log(this.arcs);
    },
    renderAxes: function() {
      if (!this.xScale || !this.yScale) return;

      const xAxis = d3
        .axisBottom()
        .scale(this.xScale)
        .tickFormat(this.format);
      const yAxis = d3
        .axisLeft()
        .scale(this.yScale)
        .tickSizeOuter(0)
        .tickFormat(d => `${d3.format("$")(parseInt(d / 1000000))}M`);

      // call axes on group elements
      d3.select(this.$refs.xAxis).call(xAxis);
      d3.select(this.$refs.yAxis)
        .call(yAxis)
        .selectAll("path")
        .remove();
    }
  }
};
</script>

<style>
.areachart {
  position: relative;
}

.arc {
  cursor: pointer;
}
</style>
