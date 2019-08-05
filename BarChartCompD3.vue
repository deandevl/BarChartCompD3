<template>
  <div class="barChartCompD3">
    <section :class="[scaling_is_active ? 'barChartCompD3_scaling' : 'barChartCompD3_not_scaling']">
      <input-comp
          heading="y minimum"
          header_position="below"
          placeholder="Enter min"
          input_size="7"
          :input_value="min"
          :single_border="single_border"
          v-on:input_comp_value_changed="value => {this.min = value}">
      </input-comp>
      <input-comp
          heading="y maximum"
          header_position="below"
          placeholder="Enter max"
          input_size="7"
          :input_value="max"
          :single_border="single_border"
          v-on:input_comp_value_changed="value => {this.max = value}">
      </input-comp>
      <input-comp
          heading="y step"
          header_position="below"
          placeholder="Enter step"
          input_size="7"
          :input_value="step"
          :single_border="single_border"
          v-on:input_comp_value_changed="value => {this.step = value}">
      </input-comp>
      <input-comp
          heading="y tick format"
          header_position="below"
          placeholder="Enter tick format"
          input_size="14"
          :input_value="tick_format"
          :single_border="single_border"
          v-on:input_comp_value_changed="value => {this.tick_format = value}">
      </input-comp>
      <button-comp
          :css_variables="button_css_variables"
          v-on:button_comp_clicked="update_scaling_axis">Update
      </button-comp>
      <button-comp
          :css_variables="button_css_variables"
          v-on:button_comp_clicked="reset_scaling_axis">Reset
      </button-comp>
    </section>
    <section class="barChartCompD3_svg__sec">
      <slot></slot>
    </section>
  </div>
</template>

<script>
  import ButtonComp from 'buttoncomp';
  import InputComp from 'inputcomp';
  import {define_scale,minmax,calculate_scale,color_scale} from 'd3scalemodule';
  import {define_axis,update_axis} from 'd3axismodule';
  import {make_draggable,init_chart} from 'd3utilmodule';
  import {format} from 'd3-format';
  import {select,local} from 'd3-selection';
  import {stack} from 'd3-shape';
  import {max,range} from 'd3-array';

  export default {
    name: 'BarChartCompD3',
    data: function() {
      return {
        data_stack: null,
        min: null,
        max: null,
        step: null,
        tick_format: null,
        current_min: null,
        current_max: null,
        current_step: null,
        current_tick_format: null,
        initials: {min: null, max: null, step: null, tick_values: null, tick_format: null},
        x_values: null,
        x_scale: null,
        y_scale: null,
        group_keys: null,
        group_scale: null,
        group_color_scale: null,
        axis_y: null,
        pixels: {x: null, y: null},
        chart_g: null,
        tooltip_text: null,
        local: local,
        scaling_is_active: false,
        alocal: null,

        button_css_variables: {
          button_comp_font_size: '.6rem'
        },

        //input-comp stuff
        single_border: true,
      };
    },
    props: {
      chart_data: {
        type: Array,
        default: null
      },
      axis: {
        type: Object,
        default: null
      },
      title_1: {
        type: String,
        default: null
      },
      title_2: {
        type: String,
        default: null
      },
      stack_bars: {
        type: Boolean,
        default: false
      },
      margin_left: {
        type: Number,
        default: 80
      },
      margin_bottom: {
        type: Number,
        default: 50
      },
      css_variables: {
        type: Object,
        default: null
      }
    },

    components: {
      ButtonComp,
      InputComp
    },
    watch: {
      chart_data: function(new_data){
        if(new_data !== null){
          //remove all current elements
          this.remove_elements();

          //define scaling/axis for x, y
          this.define_x_scaling_axis();
          this.define_y_scaling_axis();

          this.group_keys = this.axis.y.groups.map(d => {
            return d.key;
          });

          //draw bars
          this.draw_bars();

          //draw axis titles
          this.draw_axis_titles();

          //draw main titles
          this.draw_main_titles();

          //draw legend
          this.draw_legend();

          //create text toottip
          this.draw_tooltip();
        }
      }
    },
    methods: {
      define_x_scaling_axis: function(){
        //define x_scale range/domain
        this.x_values = this.chart_data.map(d => {
          return d[this.axis.x.key];
        });
        this.x_scale = define_scale(
          'band',
          'x',
          this.x_values,
          this.pixels.x,
          false,
          {paddingInner: 0.1});

        //define x axis
        define_axis(
          'bottom',
          this.axis.x.data_type,
          'barChartCompD3_axis barChartCompD3_axis_x',
          this.x_scale,
          this.chart_g,
          this.pixels.y);

        //address x tic label placement and rotation
        let x_tic_rotation = 0.0;
        let x_tic_dx = 0.0;
        let x_tic_dy = 0.6;
        let x_tic_anchor = 'middle';
        if(this.axis.x.rotation !== null && this.axis.x.rotation !== 0){
          x_tic_rotation = this.axis.x.rotation;
          x_tic_dx = -0.8;
          x_tic_dy = 0.15;
          x_tic_anchor = 'end';
        }
        this.chart_g.select('.barChartCompD3_axis_x')
          .selectAll('text')
          .attr('text-anchor', x_tic_anchor)
          .attr('dx', `${x_tic_dx}em`)
          .attr('dy', `${x_tic_dy}em`)
          .attr('transform', `rotate(${x_tic_rotation})`);
      },
      define_y_scaling_axis: function(){
        //define y_scale range/domain (linear)
        //compute overall min/max of data
        const group_keys = this.axis.y.groups.map(d => {
          return d.key;
        });
        let y_min_max = minmax(this.chart_data, group_keys);
        if(this.stack_bars){
          const astack = stack().keys(group_keys);
          this.data_stack = astack(this.chart_data);
          const values = [];
          this.data_stack[group_keys.length - 1].map(d => {
            values.push(d[1]);
          });
          y_min_max[1] = max(values);
        }

        const nice_scaling = calculate_scale(y_min_max[0], y_min_max[1]);

        //update currents
        this.current_min = nice_scaling.min;
        this.current_max = nice_scaling.max;
        this.current_step = nice_scaling.step;
        this.current_tick_format = this.axis.y.tic_format;

        //define nice y scaling
        this.y_scale = define_scale(
          'linear',
          'y',
          [nice_scaling.min, nice_scaling.max],
          this.pixels.y);

        //define tick_values
        const tick_values = range(
          this.y_scale.domain()[0],
          this.y_scale.domain()[1],
          nice_scaling.step
        );

        //define y axis
        this.axis_y = define_axis(
          'left',
          'linear',
          'barChartCompD3_axis barChartCompD3_axis_y',
          this.y_scale,
          this.chart_g,
          this.pixels.y,
          this.axis.y.tic_format,
          tick_values
        );

        //set initials
        this.initials.min = nice_scaling.min;
        this.initials.max = nice_scaling.max;
        this.initials.step = nice_scaling.step;
        this.initials.tick_values = tick_values;
        this.initials.tick_format = this.axis.y.tic_format;

        // define group_scale range/domain
        this.group_scale = define_scale(
          'band',
          'x',
          group_keys,
          this.x_scale.bandwidth(),
          false,
          {padding: 0.2});

        //define group color scaling
        const colors = this.axis.y.groups.map(d => {
          return d.color;
        });
        this.group_color_scale = color_scale(colors);

        //add click event to y axis
        this.chart_g.select('.barChartCompD3_axis_y')
          .on('click', () => {
            this.min = this.current_min;
            this.max = this.current_max;
            this.step = this.current_step;
            this.tick_format = this.current_tick_format;
            this.scaling_is_active = !this.scaling_is_active;
          });
      },
      update_scaling_axis: function(){
        const min = parseFloat(this.min);
        const max = parseFloat(this.max);
        const step = parseFloat(this.step);
        const tick_values = range(min, max, step);

        //redefine y scale domain
        this.y_scale.domain([min, max]);

        //update y axis
        const y_axis_group = this.chart_g.select('.barChartCompD3_axis_y');
        update_axis(
          this.axis_y,
          y_axis_group,
          this.y_scale,
          'linear',
          this.tick_format,
          tick_values
        );

        //update current min, max, tic
        this.current_min = min;
        this.current_max = max;
        this.current_step = step;
        this.current_tick_format = this.tick_format;

        //with the new y_scale.domain defined, redraw bars
        this.redraw_bars();
      },
      reset_scaling_axis: function(){
        //reset y_scale domain
        this.y_scale.domain([this.initials.min, this.initials.max]);

        //update the y axis with new y scale domain
        const y_axis_group = this.chart_g.select('.barChartCompD3_axis_y');
        update_axis(
          this.axis_y,
          y_axis_group,
          this.y_scale,
          'linear',
          this.initials.tick_format,
          this.initials.tick_values
        );

        //update y currents
        this.current_min = this.initials.min;
        this.current_max = this.initials.max;
        this.current_step = this.initials.step;
        this.current_tick_format = this.initials.tick_format;

        //update gui
        this.min = this.initials.min;
        this.max = this.initials.max;
        this.step = this.initials.step;
        this.tick_format = this.initials.tick_format;

        //with the new y_scale.domain defined, redraw bars
        this.redraw_bars();
      },
      draw_bars: function(){
        const self = this;
        this.alocal = new local();
        const group_formats = this.axis.y.groups.map(d => {
          return d.format;
        });
        const bars_g = this.chart_g.append('g')
          .attr('class', 'barChartCompD3_bars_g');
        if(!this.stack_bars){
          bars_g.selectAll('g').data(this.group_keys).enter()
            .append('g')
            .attr('class', 'barChartCompD3_group_bars_g')
            .attr('transform', d => {
              return `translate(${this.group_scale(d)},0)`;
            })
            .selectAll('rect').data(function(d,i){
            self.alocal.set(this,i);
            return self.chart_data;}).enter()
            .append('rect')
            .attr('class', 'barChartCompD3_rect')
            .attr('x', function(d) {
              return self.x_scale(d[self.axis.x.key]);
            })
            .attr('y', function(d) {
              return self.y_scale(d[self.group_keys[self.alocal.get(this)]]);
            })
            .attr('width', this.group_scale.bandwidth())
            .attr('height', function(d) {
              return self.pixels.y - self.y_scale(d[self.group_keys[self.alocal.get(this)]]);
            })
            .attr('fill', function() {
              return self.group_color_scale(self.alocal.get(this));
            })
            .on('mouseover', function(d){
              self.tooltip_text
                .attr('x', self.x_scale(d[self.axis.x.key]) + self.group_scale(self.group_keys[self.alocal.get(this)]) + 10)
                .attr('y', self.y_scale(d[self.group_keys[self.alocal.get(this)]]))
                .text(format(group_formats[self.alocal.get(this)])(d[self.group_keys[self.alocal.get(this)]]));
            })
            .on('mouseout', () => {
              this.tooltip_text.text('');
            });
        }else {
          const bar_g = bars_g.selectAll('g').data(this.x_values).enter()
            .append('g')
            .attr('class', 'barChartCompD3_main_bar_g')
            .attr('transform', (d, i) => {
              return `translate(${this.x_scale(this.x_values[i])},0)`;
            });

          bar_g.selectAll('rect').data(function(d, i){
            self.alocal.set(this,i);
            return self.group_keys.map((key, group_idx) => {
              return self.data_stack[group_idx];
            });
          }).enter()
            .append('rect')
            .attr('class', 'barChartCompD3_rect')
            .attr('y', function(d) {
              return self.y_scale(d[self.alocal.get(this)][1]);
            })
            .attr('width', this.x_scale.bandwidth())
            .attr('height', function(d) {
              return self.y_scale(d[self.alocal.get(this)][0]) - self.y_scale(d[self.alocal.get(this)][1]);
            })
            .attr('fill', (d, i) => {
              return this.group_color_scale(i);
            })
            .on('mouseover', function(d, i){
              //update .barChartCompD3_tooltip's text value
              let value = null;
              if(i === 0){
                value = d[self.alocal.get(this)][1];
              }else {
                value = d[self.alocal.get(this)][1] - d[self.alocal.get(this)][0];
              }
              self.tooltip_text
                .attr('x', self.x_scale(self.x_values[self.alocal.get(this)]) + self.x_scale.bandwidth() * .45)
                .attr('y', self.y_scale(d[self.alocal.get(this)][1]) + 10)
                .text(format(group_formats[i])(value));
            })
            .on('mouseout', () => {
              this.tooltip_text.text('');
            });
        }
      },
      redraw_bars: function(){
        //with the new y_scale redraw the rect bars
        const self = this;
        if(!this.stack_bars){
          //update rect's y and height attributes
          this.chart_g.selectAll('.barChartCompD3_rect')
            .attr('y', function(d) {
              return self.y_scale(d[self.group_keys[self.alocal.get(this)]]);
            })
            .attr('height', function(d) {
              const height = self.pixels.y - self.y_scale(d[self.group_keys[self.alocal.get(this)]]);
              if(height < 0.0){
                return 0;
              }else{
                return height;
              }
            });
        }else{
          //move the base up with possible new minimum value
          for(let i=0; i<this.data_stack[0].length; i++){
            this.data_stack[0][i][0] = this.y_scale.domain()[0];
          }
          //update rect's y and height attributes
          this.chart_g.selectAll('.barChartCompD3_rect')
            .attr('y', function(d){
              return self.y_scale(d[self.alocal.get(this)][1]);
            })
            .attr('height', function(d){
              return self.y_scale(d[self.alocal.get(this)][0]) - self.y_scale(d[self.alocal.get(this)][1]);
            });
        }
      },
      draw_axis_titles: function(){
        //draw x axis title
        const axis_title_g_x = this.chart_g.append('g')
          .attr('class', 'barChartCompD3_axis_title barChartCompD3_axis_title_g_x');
        axis_title_g_x.append('text')
          .attr('x', this.pixels.x / 3)
          .attr('y', this.pixels.y + this.margin_bottom - 10)
          .text(this.axis.x.title);
        make_draggable(axis_title_g_x);

        //draw y axis title
        const axis_title_g_y = this.chart_g.append('g')
          .attr('class', 'barChartCompD3_axis_title barChartCompD3_axis_title_g_y');
        axis_title_g_y.append('text')
          .attr('transform', 'rotate(-90)')
          .attr('x', -0.67 * this.pixels.y)
          .attr('y', -this.margin_left + 30)
          .text(this.axis.y.title);
        make_draggable(axis_title_g_y);
      },
      draw_main_titles: function(){
        const title_1_g = this.chart_g.append('g')
          .attr('class','barChartCompD3_title_g_1');
        title_1_g.append('text')
          .attr('class','barChartCompD3_title')
          .attr('transform',`translate(${this.margin_left+this.pixels.x/2},30)`)
          .attr('text-anchor','middle')
          .style('font-size','1.3rem')
          .text(this.title_1);
        make_draggable(title_1_g);

        const title_2_g = this.chart_g.append('g')
          .attr('class','barChartCompD3_title_g_2');
        title_2_g.append('text')
          .attr('class','barChartCompD3_title')
          .attr('transform',`translate(${this.margin_left+this.pixels.x/2},50)`)
          .attr('text-anchor','middle')
          .style('font-size','.8rem')
          .text(this.title_2);
        make_draggable(title_2_g);
      },
      draw_legend: function(){
        if(this.group_keys.length > 1){
          const legends_g = this.chart_g.append('g')
            .attr('class', 'barChartCompD3_legends_g');
          const legend = legends_g.selectAll('g').data(this.group_keys).enter()
            .append('g')
            .attr('transform', (d, i) => {
              return `translate(0,${i * 20})`;
            });
          legend.append('rect')
            .attr('class', 'barChartCompD3_rect_legend')
            .attr('x', this.pixels.x - 19)
            .attr('width', 19)
            .attr('height', 19)
            .attr('fill', (d, i) => {
              return this.group_color_scale(i);
            });
          legend.append('text')
            .attr('x', this.pixels.x - 24)
            .attr('y', 9.5)
            .attr('dy', '0.32em')
            .attr('font-family', 'Verdana')
            .attr('font-size', 10)
            .attr('text-anchor', 'end')
            .text(d => {
              return d;
            });
          //make legend draggable
          make_draggable(legends_g);
        }
      },
      draw_tooltip: function(){
        this.tooltip_text = this.chart_g.append('text')
          .attr('class', 'barChartCompD3_tooltip')
          .attr('x',0)
          .attr('y',0)
          .text('');
      },
      remove_elements: function(){
        //remove x axis
        this.chart_g.select('.barChartCompD3_axis_x').remove();
        //remove y axis
        this.chart_g.select('.barChartCompD3_axis_y').remove();
        //remove bars
        this.chart_g.select('.barChartCompD3_bars_g').remove();
        //remove axis titles
        this.chart_g.select('.barChartCompD3_axis_title_g_x').remove();
        this.chart_g.select('.barChartCompD3_axis_title_g_y').remove();
        //remove main titles
        this.chart_g.select('.barChartCompD3_title_g_1').remove();
        this.chart_g.select('.barChartCompD3_title_g_2').remove();
        //remove legends
        this.chart_g.select('.barChartCompD3_legends_g').remove();
        //remove tooltip
        this.chart_g.select('.barChartCompD3_tooltip').remove();
      }
    },
    mounted: function() {
      //locate svg
      const svg_class = this.$slots.default[0].elm.classList[0];
      const svg = select(`.${svg_class}`);

      const chart = init_chart(svg, this.margin_left, undefined, undefined, this.margin_bottom);
      this.pixels.x = chart.chart_width;
      this.pixels.y = chart.chart_height;
      this.chart_g = chart.chart_g;

      if(this.css_variables !== null){
        for(let key of Object.keys(this.css_variables)){
          this.$el.style.setProperty(`--${key}`, this.css_variables[key]);
        }
      }
    }
  }
</script>

<style>
  :root {
    --bar_chart_compD3_font_family: Verdana,serif;
    --bar_chart_compD3_color: black;
    --bar_chart_compD3_background_color: white;
    --bar_chart_compD3_axis_font_size: 0.8rem;
    --bar_chart_compD3_axis_color: black;
    --bar_chart_compD3_rect_stroke: black;
    --bar_chart_compD3_rect_fill_opacity: 1;
    --bar_chart_compD3_tooltip_fill: black;
  }
  .buttonComp {
    align-self: center;
  }
  .barChartCompD3 {
    display: flex;
    flex-direction: column;
    font-family: var(--bar_chart_compD3_font_family);
    color: var(--bar_chart_compD3_color);
    background-color: var(--bar_chart_compD3_background_color);
  }
  .barChartCompD3_title {
    font-weight: bold;
    fill: var(--bar_chart_compD3_color);
    cursor: move;
  }
  .barChartCompD3_rect {
    stroke: var(--bar_chart_compD3_rect_stroke);
    fill-opacity: var(--bar_chart_compD3_rect_fill_opacity);
    cursor: pointer;
  }
  .barChartCompD3_rect_legend {
    stroke: var(--bar_chart_compD3_rect_stroke);
    fill-opacity: var(--bar_chart_compD3_rect_fill_opacity);
  }
  .barChartCompD3_axis_title {
    font-weight: bold;
    fill: var(--bar_chart_compD3_axis_color);
    cursor: move;
  }
  .barChartCompD3_axis_y {
    cursor: pointer;
  }
  .barChartCompD3_axis path,
  .barChartCompD3_axis line {
    shape-rendering: crispEdges;
    color: var(--bar_chart_compD3_axis_color);
  }
  .barChartCompD3_axis text {
    font-weight: bold;
    font-family: var(--bar_chart_compD3_font_family);
    font-size: var(--bar_chart_compD3_axis_font_size);
    color: var(--bar_chart_compD3_axis_color);
  }
  .barChartCompD3_scaling {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    width: 600px;
    margin-bottom: 5px;
  }
  .barChartCompD3_not_scaling {
    display: none;
    border: none;
    margin: 0;
    text-anchor: middle;
    font-weight: bold;
    font-size: 12px;
    font-family: var(--bar_chart_compD3_font_family);
    fill: var(--bar_chart_compD3_tooltip_fill);
  }
  .barChartCompD3_tooltip {
    text-anchor: middle;
    font-weight: bold;
    font-size: 12px;
    font-family: var(--bar_chart_compD3_font_family);
    fill: var(--bar_chart_compD3_tooltip_fill);
  }
  .barChartCompD3_legends_g {
    cursor: move;
  }
</style>