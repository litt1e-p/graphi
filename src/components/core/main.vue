<template>
  <div class="graphi">
    <svg class="svg" v-bind:width="conf.width" v-bind:height="conf.height"></svg>
  </div>
</template>

<script>

const d3 = require('d3')

export default {
  name: 'graphi',
  props: {
    value: {
      type: Object,
      default: () => void 0
    },
    conf: {
      type: Object,
      default: () => {
        return {
          width: 0,
          height: 0,
          zoomToFit: true,
          fitScale: 0.6,
          bubbleRadius: 60,
          arrowOffset: 0,
          relations: void 0
        }
      }
    }
  },
  data() {
    return {
      force: void 0,
      nodes: Array.prototype.constructor(),
      edges: Array.prototype.constructor(),
      links: void 0,
      linksText: void 0,
      linksTextPath: void 0,
      gs: void 0,
      svg: void 0,
      g: void 0,
      zoom: void 0,
      dto: void 0
    }
  },
  mounted() {
    this.build()
  },
  methods: {
    trueOfalse (o, k, cb) {
      if (!o) {
        return false
      }
      if (o.hasOwnProperty(k) && !cb(o[k])) {
        return false
      }
      return true
    },
    preBuild (v) {
      if (!v || !v['name'] || v['childs'].constructor !== Array) {
        return false
      }
      if (!this.trueOfalse(this.conf, 'width', (i) => /\d+/.test(i) && typeof i === 'number')) {
        return false
      }
      if (!this.trueOfalse(this.conf, 'height', (i) => /\d+/.test(i) && typeof i === 'number')) {
        return false
      }
      if (!this.trueOfalse(this.conf, 'bubbleRadius', (i) => /\d+/.test(i) && typeof i === 'number')) {
        return false
      }
      if (!this.trueOfalse(this.conf, 'arrowOffset', (i) => /\d+/.test(i) && typeof i === 'number')) {
        return false
      }
      if (!this.trueOfalse(this.conf, 'zoomToFit', (i) => typeof i === 'boolean')) {
        return false
      }
      return true
    },
    build (v) {
      v = v || this.value
      if (!this.preBuild(v)) {
        return
      }
      this.dto = v
      this.assemble(this.dto)
      if (!this.svg || !this.force) {
        this.initGraph()
      }
      this.update()
    },
    assemble (d) {
      if (!d) {
        return
      }
      let ns = Array.prototype.constructor()
      let root = {...this.vencrt('name', d['name']), ...{'rel': 'origin'}}
      ns.push(root)
      ns = ns.concat(this.noderix(d['childs'], 'childs'))
      ns = ns.concat(this.noderix(d['members'], 'members'))
      ns = ns.concat(this.noderix(d['outsides'], 'outsides'))
      let es = Array.prototype.constructor()
      for (let i = 0; i < ns.length; i++) {
        const n = ns[i]
        if (i === 0) {
          continue
        }
        es.push(this.vencrts(i, n))
      }
      this.nodes = ns
      this.edges = es
    },
    noderix (d, rel) {
      let nr = Array.prototype.constructor()
      for (let i = 0; i < d.length; i++) {
        const c = d[i]
        nr.push({...this.vencrt('name', c['name']), ...this.vencrt('rel', rel), ...this.vencrt('data', c)})
      }
      return nr
    },
    vencrts (i, d, root = 0) {
      let v = ({'childs': this.random(3, 4), 'members': this.random(2, 3), 'outsides': this.random(2, 4)})[d['rel']]
      return {...this.vencrt('source', i), ...this.vencrt('target', root), ...this.vencrt('value', v), ...this.vencrt('relation', d['rel'])}
    },
    vencrt (k, v) {
      let o = Object.create(null)
      o[k] = v
      return o
    },
    sizeToFit () {
      let scale = this.conf.fitScale || 0.6
      this.svg.transition()
        .duration(750)
        .call(this.zoom.transform,
          d3.zoomIdentity
            .translate(this.resize(scale).zoomWidth, this.resize(scale).zoomHeight)
            .scale(scale)
      )
    },
    initGraph () {
      this.conf.width = this.conf.width || this.$el.clientWidth
      this.conf.height = this.conf.height || 500
      let rect = {top: 60, bottom: 60, left: 60, right: 60}
      let zoom = d3.zoom()
        .scaleExtent([0.2, 8])
        .on('zoom', this.trans)
      this.svg = d3.select('.svg')
        .call(zoom)
        .on('dblclick.zoom', null)
      this.zoom = zoom
      if (this.conf.zoomToFit) {
        this.sizeToFit()
      }
      this.g = this.svg.append('g').attr('transform', `translate(${rect.top},${rect.left})`)
      this.force = d3.forceSimulation()
        .force('charge', d3.forceManyBody().strength(-900))
        .force('link', d3.forceLink().distance(50))
        .force('center', d3.forceCenter())
    },
    tick () {
      if (!this.links || !this.linksText || !this.gs) {
        return
      }
      this.links
        .attr('x1', function (d) { return d.source.x })
        .attr('y1', function (d) { return d.source.y })
        .attr('x2', function (d) { return d.target.x })
        .attr('y2', function (d) { return d.target.y })
      this.linksTextPath.attr('d', function (d) {
        return 'M ' + d.source.x + ' ' + (d.source.y - 5) + ' L ' + d.target.x + ' ' + d.target.y
      })
      this.linksText.attr('transform', function (d) {
        if (d.target.x < d.source.x) {
          let bbox = this.getBBox()
          let rx = bbox.x + bbox.width / 2
          let ry = bbox.y + bbox.height / 2
          return 'rotate(180 ' + rx + ' ' + ry + ')'
        }
        return 'rotate(0)'
      })
      this.gs
        .attr('transform', function (d) { return 'translate(' + d.x + ',' + d.y + ')' })
    },
    _getRel(d) {
      if (this.conf.relations && Object.keys(this.conf.relations).length > 0) {
        const rf = this.conf.relations[d['relation']]
        if (Object.prototype.toString.call(rf) === '[object String]') {
          return rf
        } else if (Object.prototype.toString.call(rf) === '[object Array]') {
          const k = d.source.data.key || 'id'
          const tg = rf.find(i => i[k] === d.source.data[k])
          if (tg) {
            return tg['name']
          }
        }
      }
      return d['relation']
    },
    update () {
      let colorScale = d3.scaleOrdinal()
        .domain(d3.range(this.nodes.length))
        .range(d3.schemeCategory10)
      colorScale = function (d) {
        return ({'origin': '#8788D3', 'childs': '#86C99E', 'members': '#CFD0EE', 'outsides': '#FDD495'})[d['rel']]
      }
      this.force.nodes(this.nodes).on('tick', this.tick)
      this.force.force('link')
        .links(this.edges)
        .distance(function (d) {
          return d.value * 100
        })
      this.force.force('center')
        .x(this.conf.width / 2)
        .y(this.conf.height / 2)
      let linkG = this.g.append('g')
      this.links = linkG
        .selectAll('line')
        .data(this.edges)
        .enter()
        .append('line')
        .attr('stroke', function (d, i) {
          return '#ccc'
        })
        .attr('class', function (d) { return d.relation === 'members' ? 'link link_dashed' : 'link link_continuous' })
        .attr('stroke-width', 1)
        .attr('marker-start', 'url(#arrow)')
      linkG.append('defs').append('marker')
        .attr('id', 'arrow')
        .attr('viewBox', '0 -5 20 20')
        // .attr('refX', -53)
        .attr('refX', (d) => {
          return -(this.conf.bubbleRadius + 15 + (this.conf.arrowOffset || 0))
        })
        .attr('refY', 0)
        .attr('markerWidth', 15)
        .attr('markerHeight', 15)
        .attr('orient', 'auto')
        .append('path')
        .attr('d', 'M0,0L10,-5L10,5')
        .attr('fill', '#ccc')
        .attr('class', 'arrowhead')

      let linksTextG = this.g.append('g')
      this.linksTextPath = linksTextG.selectAll('.edgepath')
        .data(this.edges)
        .enter()
        .append('path')
        .attr('class', 'edgepath')
        .attr('fill-opacity', 0)
        .attr('stroke-opacity', 0)
        .attr('id', function (d, i) { return 'edgepath' + i })
        .style('pointer-events', 'none')
      this.linksText = linksTextG
        .selectAll('.edgelabel')
        .data(this.edges)
        .enter()
        .append('text')
        .style('pointer-events', 'none')
        .attr('class', 'edgelabel')
        .attr('id', function (d, i) { return 'edgelabel' + i })
        .attr('font-size', 10)
        .attr('fill', '#aaa')

      this.linksText.append('textPath')
        .data(this.edges)
        .attr('xlink:href', function (d, i) { return '#edgepath' + i })
        .style('text-anchor', 'middle')
        .style('pointer-events', 'none')
        .attr('startOffset', '50%')
        .text(d => this._getRel(d))

      this.gs = this.g.selectAll('.circleText')
        .data(this.nodes)
        .enter()
        .append('g')
        .attr('transform', function (d, i) {
          return 'translate(' + d.x + ',' + d.y + ')'
        })
        .call(d3.drag()
          .on('start', this.started)
          .on('drag', this.dragged)
          .on('end', this.ended)
        )
      this.gs.append('circle')
        .attr('r', (d) => {
          return d.index === 0 ? this.conf.bubbleRadius + 10 : 0
        })
        .attr('fill', function (d, i) {
          return '#D1D2EE'
        })
      this.gs.append('circle')
        .attr('r', this.conf.bubbleRadius)
        .attr('fill', function (d, i) {
          return colorScale(d)
        })
      this.gs.append('text')
        .attr('x', 0)
        .style('text-anchor', 'middle')
        .style('font-weight', 'normal')
        .style('pointer-events', 'none')
        .style('fill', function (d) {
          return ({'origin': '#FFFFFF', 'childs': '#FFFFFF', 'members': '#8788D3', 'outsides': '#B4843A'})[d['rel']]
        })
        .attr('dy', '.35em')
        .text(function (d) {
          return d.name
        })
        .call(this.wrap)
    },
    wrap (text) {
      let _ = this
      text.each(function () {
        let width = _.conf.bubbleRadius * 2
        let text = d3.select(this)
        let originText = text.text()
        let words = originText.split('').reverse()
        let word
        let line = []
        let lineNumber = 0
        let lineHeightBase = 1.0
        let lineHeight = 0.1
        let x = text.attr('x')
        let y = text.attr('y')
        let dy = 0
        let tspan = text.text(null)
          .append('tspan')
          .attr('x', x)
          .attr('y', y)
          .attr('dy', dy + 'em')
        // eslint-disable-next-line no-cond-assign
        while (word = words.pop()) {
          line.push(word)
          tspan.text(line.join(''))
          if (tspan.node().getComputedTextLength() > width) {
            line.pop()
            tspan.text(line.join(''))
            line = [word]
            tspan = text.append('tspan')
              .attr('x', x)
              .attr('y', y)
              .attr('dy', lineHeightBase + ++lineNumber * lineHeight + dy + 'em')
              .text(word)
          }
        }
        text.attr('alignment-baseline', 'central')
        text.attr('dominant-baseline', 'central')
      })
    },
    trans () {
      this.g.attr('transform', d3.event.transform)
    },
    resize (scale = 1) {
      return this.getSize(scale)
    },
    getSize (scale) {
      let width = this.conf.width || this.$el.clientWidth
      let height = this.conf.height || this.$el.clientHeight
      let zoomWidth = (width - scale * width) / 2
      let zoomHeight = (height - scale * height) / 2
      return { zoomWidth, zoomHeight }
    },
    started (d) {
      if (!d3.event.active) {
        this.force.alphaTarget(0.8).restart()
      }
      d.fx = d.x
      d.fy = d.y
    },
    dragged (d) {
      d.fx = d3.event.x
      d.fy = d3.event.y
    },
    ended (d) {
      if (!d3.event.active) {
        this.force.alphaTarget(0)
      }
      d.fx = null
      d.fy = null
    },
    random (min, max) {
      return Math.random() * (max - min) + min
    }
  },
  watch: {
    value (v) {
      this.build(v)
    }
  },
}
</script>

<style lang="scss" scoped>
.node {
  cursor: pointer;
  circle {
    stroke: #34495e;
    stroke-width: 2px;
    box-sizing: border-box;
    stroke-location: inside;
  }
  text {
    font-size: 11px;
  }
}

line.link {
  fill: none;
  stroke: #bbb;
  stroke-width: 1.5px;
}
>>> .link_continuous {
  fill: none;
  stroke: #bbb;
  stroke-width: 1.5px;
}
>>> .link_dashed {
  fill: none;
  stroke: #bbb;
  stroke-width: 1.5px;
  stroke-dasharray: 5,5,5,5,5,5;
}
>>> .arrow{
  stroke-width:5;
  stroke:#000;
  stroke-dasharray:5, 5;
}

>>> #arrow{
  stroke-width:1;
  stroke-dasharray:0;
}
</style>
