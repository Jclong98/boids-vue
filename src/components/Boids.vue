<script setup>
import { ref, onMounted, computed } from 'vue'
import { useElementSize, useRafFn, useMouse } from '@vueuse/core'

const debug = ref(false)
const boidCount = ref(180)
const visionRadius = ref(100)
const personalSpaceRadius = ref(20)
const maxSpeed = ref(0.8)
const maxAlignmentForce = ref(0.003)
const maxCohesionForce = ref(0.001)
const maxSeparationForce = ref(0.004)
const maxMouseForce = ref(0.0)
const color = ref('tomato')
const randomColors = ref(true)
const drawAsTriangles = ref(false)

const canvasParent = ref()
const { width, height } = useElementSize(canvasParent)
const mouse = useMouse()

const canvasElement = ref()
const ctx = ref()

class Vector {
  constructor(x, y) {
    this.x = x
    this.y = y
  }

  set(x, y) {
    this.x = x
    this.y = y
  }

  add(vector) {
    this.x += vector.x
    this.y += vector.y
  }

  subtract(vector) {
    this.x -= vector.x
    this.y -= vector.y
  }

  multiply(scalar) {
    this.x *= scalar
    this.y *= scalar
  }

  divide(scalar) {
    this.x /= scalar
    this.y /= scalar
  }

  distance(vector) {
    return Math.sqrt(
      Math.pow(this.x - vector.x, 2) + Math.pow(this.y - vector.y, 2)
    )
  }

  get magnitude() {
    return Math.sqrt(Math.pow(this.x, 2) + Math.pow(this.y, 2))
  }

  setMagnitude(magnitude) {
    const currentMagnitude = this.magnitude
    if (currentMagnitude === 0) {
      return
    }
    this.multiply(magnitude / currentMagnitude)
  }

  limit(maxMagnitude) {
    if (this.magnitude > maxMagnitude) {
      this.setMagnitude(maxMagnitude)
    }
  }
}

class Boid {
  constructor(x, y) {
    this.position = new Vector(x, y)
    this.velocity = new Vector(Math.random() * 2 - 1, Math.random() * 2 - 1)
    this.velocity.setMagnitude(maxSpeed.value)
    this.acceleration = new Vector(0, 0)
    this.assignedColor = `hsl(${Math.random() * 360}, 100%, 50%)`
  }

  update(boids) {
    this.wrap()

    this.position.add(this.velocity)
    this.velocity.add(this.acceleration)

    this.acceleration.add(this.getAlignmentForce(boids))
    this.acceleration.add(this.getCohesionForce(boids))
    this.acceleration.add(this.getSeparationForce(boids))
    this.acceleration.add(this.getMouseForce(boids))

    this.acceleration.limit(maxSpeed.value)
    this.velocity.limit(maxSpeed.value)

    this.draw()
  }

  getMouseForce() {
    const steeringForce = new Vector(0, 0)
    const mouseVector = new Vector(mouse.x.value, mouse.y.value)

    steeringForce.add(mouseVector)
    steeringForce.subtract(this.position)
    steeringForce.limit(maxSpeed.value)
    steeringForce.subtract(this.velocity)
    steeringForce.limit(maxMouseForce.value)

    return steeringForce
  }

  getSeparationForce(boids) {
    // get boids within the vision radius
    const nearbyBoids = boids.filter((boid) => {
      if (boid === this) return false // ignore self
      const distance = this.position.distance(boid.position)
      return distance < personalSpaceRadius.value
    })

    // get the average position of all nearby boids
    const steeringForce = new Vector(0, 0)
    nearbyBoids.forEach((boid) => {
      const difference = new Vector(
        this.position.x - boid.position.x,
        this.position.y - boid.position.y
      )
      difference.multiply(1 / this.position.distance(boid.position))
      steeringForce.add(difference)
    })

    if (nearbyBoids.length > 0) {
      steeringForce.divide(nearbyBoids.length)
      steeringForce.limit(maxSpeed.value)
      steeringForce.subtract(this.velocity)
      steeringForce.limit(maxSeparationForce.value)
    }

    return steeringForce
  }

  getCohesionForce(boids) {
    // get boids within the vision radius
    const nearbyBoids = boids.filter((boid) => {
      if (boid === this) return false // ignore self
      const distance = this.position.distance(boid.position)
      return distance < visionRadius.value
    })

    // get the average position of all nearby boids
    const steeringForce = new Vector(0, 0)
    nearbyBoids.forEach((boid) => {
      steeringForce.add(boid.position)
    })

    if (nearbyBoids.length > 0) {
      steeringForce.divide(nearbyBoids.length)
      steeringForce.subtract(this.position)
      steeringForce.limit(maxSpeed.value)
      steeringForce.subtract(this.velocity)
      steeringForce.limit(maxCohesionForce.value)
    }

    return steeringForce
  }

  getAlignmentForce(boids) {
    // get boids within the vision radius
    const nearbyBoids = boids.filter((boid) => {
      if (boid === this) return false // ignore self
      const distance = this.position.distance(boid.position)
      return distance < visionRadius.value
    })

    // draw a line to each boid in debug
    if (debug.value) {
      nearbyBoids.forEach((boid) => {
        ctx.value.strokeStyle = 'lightgray'
        ctx.value.beginPath()
        ctx.value.moveTo(this.position.x, this.position.y)
        ctx.value.lineTo(boid.position.x, boid.position.y)
        ctx.value.stroke()
      })
    }

    // find the average position of all nearby boids
    const steeringForce = new Vector(0, 0)
    for (let other of nearbyBoids) {
      steeringForce.add(other.velocity)
    }

    if (nearbyBoids.length > 0) {
      steeringForce.divide(nearbyBoids.length)
      steeringForce.setMagnitude(maxSpeed.value)
      steeringForce.subtract(this.velocity)
      steeringForce.limit(maxAlignmentForce.value)
    }

    return steeringForce
  }

  /**
   * warp the boid to the other side of the screen if it goes out of bounds
   */
  wrap() {
    if (this.position.x < 0) {
      this.position.x = width.value
    } else if (this.position.x > width.value) {
      this.position.x = 0
    }

    if (this.position.y < 0) {
      this.position.y = height.value
    } else if (this.position.y > height.value) {
      this.position.y = 0
    }
  }

  /**
   * Draw the boid on the canvas
   * include debug info if debug is true
   */
  draw() {
    if (!ctx.value) return

    ctx.value.fillStyle = randomColors.value ? this.assignedColor : color.value

    if (drawAsTriangles.value) {
      // draw a triangle in the direction of the boid's velocity
      ctx.value.beginPath()
      ctx.value.moveTo(
        this.position.x + this.velocity.x * 20,
        this.position.y + this.velocity.y * 20
      )
      ctx.value.lineTo(
        this.position.x - this.velocity.y * 10,
        this.position.y + this.velocity.x * 10
      )
      ctx.value.lineTo(
        this.position.x + this.velocity.y * 10,
        this.position.y - this.velocity.x * 10
      )
      ctx.value.lineTo(
        this.position.x + this.velocity.x * 20,
        this.position.y + this.velocity.y * 20
      )
      ctx.value.fill()
    } else {
      // draw a circle at the boid's position
      ctx.value.beginPath()
      ctx.value.arc(this.position.x, this.position.y, 5, 0, Math.PI * 2)
      ctx.value.fill()
    }

    if (!debug.value) return

    // draw a line from the boid's position to the boid's velocity
    ctx.value.strokeStyle = 'black'
    ctx.value.beginPath()
    ctx.value.moveTo(this.position.x, this.position.y)
    ctx.value.lineTo(
      this.position.x + this.velocity.x * 10,
      this.position.y + this.velocity.y * 10
    )
    ctx.value.stroke()

    // position
    ctx.value.fillText(
      `position: (${this.position.x.toFixed()}, ${this.position.y.toFixed()})`,
      this.position.x - 10,
      this.position.y - 10
    )

    // show vision radius
    ctx.value.beginPath()
    ctx.value.arc(
      this.position.x,
      this.position.y,
      visionRadius.value,
      0,
      Math.PI * 2
    )
    ctx.value.strokeStyle = 'lightgray'
    ctx.value.stroke()

    // show personal space radius
    ctx.value.beginPath()
    ctx.value.arc(
      this.position.x,
      this.position.y,
      personalSpaceRadius.value,
      0,
      Math.PI * 2
    )
    ctx.value.strokeStyle = 'tomato'
    ctx.value.stroke()
  }
}

const boids = computed(() => {
  const arr = []
  for (let i = 0; i < boidCount.value; i++) {
    const posX = Math.random() * width.value
    const posY = Math.random() * height.value

    arr.push(
      new Boid(posX, posY)
      // new Boid(width.value / 2, height.value / 2)
    )
  }
  return arr
})

const { pause, resume } = useRafFn(() => {
  // draw a green circle in the center of the canvas
  if (!ctx.value) return
  ctx.value.clearRect(0, 0, width.value, height.value)

  // update the position of each boid
  boids.value.forEach((boid) => boid.update(boids.value))
})

onMounted(() => {
  ctx.value = canvasElement.value.getContext('2d')
})
</script>

<template>
  <details class="debug-panel">
    <summary>Conrols</summary>

    <div style="padding-top: 0.5rem">
      <label>
        <input type="checkbox" v-model="debug" />
        Show debug info
      </label>
    </div>

    <hr />

    <div>
      <label class="input-grid">
        <span>Number of boids</span>
        <input type="range" v-model="boidCount" min="1" max="500" />
        <input type="number" v-model="boidCount" />
      </label>
    </div>

    <hr />

    <div>
      <label class="input-grid">
        <span>Vision Radius</span>
        <input type="range" v-model="visionRadius" min="0" max="200" />
        <input type="number" v-model="visionRadius" />
      </label>
    </div>
    <div>
      <label class="input-grid">
        <span>Personal Space Radius</span>
        <input type="range" v-model="personalSpaceRadius" min="0" max="200" />
        <input type="number" v-model="personalSpaceRadius" />
      </label>
    </div>

    <hr />

    <div>
      <label class="input-grid">
        <span>Maxumum Speed</span>
        <input type="range" v-model="maxSpeed" min="0" max="5" step="0.1" />
        <input type="number" v-model="maxSpeed" />
      </label>
    </div>
    <div>
      <label class="input-grid">
        <span>Max Alignment Force</span>
        <input
          type="range"
          v-model="maxAlignmentForce"
          min="0"
          max="0.02"
          step="0.001"
        />
        <input type="number" v-model="maxAlignmentForce" />
      </label>
    </div>
    <div>
      <label class="input-grid">
        <span>Cohesion Force</span>
        <input
          type="range"
          v-model="maxCohesionForce"
          min="0"
          max="0.02"
          step="0.001"
        />
        <input type="number" v-model="maxCohesionForce" />
      </label>
    </div>
    <div>
      <label class="input-grid">
        <span>Separation Force</span>
        <input
          type="range"
          v-model="maxSeparationForce"
          min="0"
          max="0.05"
          step="0.001"
        />
        <input type="number" v-model="maxSeparationForce" />
      </label>
    </div>
    <div>
      <label class="input-grid">
        <span>Mouse Force</span>
        <input
          type="range"
          v-model="maxMouseForce"
          min="0"
          max="0.01"
          step="0.001"
        />
        <input type="number" v-model="maxMouseForce" />
      </label>
    </div>

    <hr />

    <div>
      <label class="input-grid">
        <span>Color</span>
        <input type="color" v-model="color" />
      </label>
    </div>

    <div class="input-grid" style="margin-top: 0.5rem">
      <label for="random-colors"> Random Colors </label>
      <input type="checkbox" id="random-colors" v-model="randomColors" />
    </div>

    <div class="input-grid" style="margin-top: 0.5rem">
      <label for="draw-as-triangles"> Draw as Triangles </label>
      <input type="checkbox" id="draw-as-triangles" v-model="drawAsTriangles" />
    </div>
  </details>

  <div ref="canvasParent" class="boids-container" style="z-index: -1">
    <canvas ref="canvasElement" :width="width" :height="height"></canvas>
  </div>
</template>

<style>
.boids-container {
  position: absolute;
  inset: 0;
  background: black;
}

.boids-container canvas {
  position: absolute;
  inset: 0;
}

.debug-panel {
  background-color: white;
  padding: 10px;
  margin: 10px;
  width: fit-content;
  border-radius: 4px;
}

.input-grid {
  display: grid;
  grid-template-columns: 1fr 1fr 0.5fr;
  grid-gap: 10px;
  text-align: right;
}

.input-grid input[type='number'] {
  width: 60px;
}
</style>
