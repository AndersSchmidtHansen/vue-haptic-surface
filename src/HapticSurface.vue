<template>
  <div v-on="$listeners" @mousemove="sense" @click="sense"><slot></slot></div>
</template>

<script>
export default {
  name: 'HapticSurface',
  props: {
    maxClickPositionsSaved: {
      type: Number,
      default: 5
    }
  },
  data () {
    return {
      memory: {
        clicks: []
      },
      events: {
        mouse: new MouseEvent(null)
      }
    }
  },
  computed: {
    context () {
      return {
        shape: this.shape,
        center: this.center,
        pointer: this.pointer,
        distance: this.distance,
        corners: this.corners,
        memory: this.memory
      }
    },
    center () {
      return this.getCoords(this.shape.width / 2, this.shape.height / 2)
    },
    distance () {
      const points = [
        {
          type: 'topLeft',
          coords: this.getCoords(this.pointer.x, this.pointer.y)           
        },
        {
          type: 'topRight',
          coords: this.getCoords(this.corners.topRight.x - this.pointer.x, this.corners.topRight.y + this.pointer.y)           
        },
        {
          type: 'bottomRight',
          coords: this.getCoords(this.corners.bottomRight.x - this.pointer.x, this.corners.bottomRight.y - this.pointer.y)           
        },
        {
          type: 'bottomLeft',
          coords: this.getCoords(this.corners.bottomLeft.x + this.pointer.x, this.corners.bottomLeft.y - this.pointer.y)           
        }            
      ]
      
      const output = points.reduce((a, b) => {
        a[b.type] = {
          ...b.coords,
          ...this.getDistanceFromCorner(b.type),
          ...this.getCornerToCenterPercentage(b.coords, this.center)
        }
        return a
      }, {
        center: {
          ...this.getCoords(this.pointer.x - this.center.x, this.center.y - this.pointer.y),
          d: this.getDistance(this.pointer.x, this.center.x, this.pointer.y, this.center.y)
        }
      })  
      return output
    },
    corners () {
      return {
        topLeft:     this.getCoords(0, 0),
        topRight:    this.getCoords(this.shape.width, 0),
        bottomRight: this.getCoords(this.shape.height, this.shape.width),
        bottomLeft:  this.getCoords(0, this.shape.height)
      }
    },
    shape () {
      const rect = this.$el.getBoundingClientRect()
      return rect
    },
    pointer () {
      const {
        offsetX: x,
        offsetY: y
      } = this.events.mouse
      return {
        ...this.getCoords(x, y),
        degrees: this.getPointDegrees(y, this.center.y, x, this.center.x)
      }
    }
  },
  methods: {
    clamp (min, max) {
      return Math.min(Math.max(0, min), max)
    },
    getCornerToCenterPercentage (corner, center, clamp = 100) {
      const clampedPct = (c, axis = 'x', dir = 'getPercentageIncrease') => {
        return this.clamp(this[dir](corner[axis], c), clamp)
      }
      return ({
        to_center_x_pct: clampedPct(center.x),
        to_center_y_pct: clampedPct(center.y, 'y'),
        from_center_x_pct: clampedPct(center.x, 'x', 'getPercentageDecrease'),
        from_center_y_pct: clampedPct(center.y, 'y', 'getPercentageDecrease')
      })
    }, 
    getDistanceFromCorner (corner = 'topLeft') {
      if (!this.pointer) return
      return this.getDistance(this.pointer.x, this.corners[corner].x, this.pointer.y, this.corners[corner].y)
    },
    getCoords (x, y) {
      return ({
        x,
        y,
        inverted_x: x < 0 ? Math.abs(x) : -x,
        inverted_y: y < 0 ? Math.abs(y) : -y
      })
    },
    getPointDegrees (y, cy, x, cx) {
      return Math.floor((Math.atan2(y - cy, x - cx) * 180/Math.PI + 360) % 360)
    },
    getPercentageIncrease (x, y) {
      return Math.floor((x / y) * 100)
    },
    getPercentageDecrease (x, y) {
      return Math.floor(((y - x) / y) * 100)
    },
    getDistance (x1, x2, y1, y2) {
      return Math.floor(Math.hypot(x2 - x1, y2 - y1))
    },
    setPointerPositionInMemory () {
      const payload = {
        ...this.getCoords(this.pointer.x, this.pointer.y),
        created_at: new Date().getTime()
      }
      this.memory.clicks = [payload, ...this.memory.clicks].slice(0, this.maxClickPositionsSaved)
    },
    sense (event) {
      if (event.type.match('mousemove')) {
        this.events.mouse = event
      }
      
      if (event.type.match('click')) {
        this.setPointerPositionInMemory()
      }
      
      this.$emit('sensing', this.context)
    }
  }  
}
</script>
