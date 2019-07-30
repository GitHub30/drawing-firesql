<template>
  <div class="container">
    <h1>FireSQL Collaborative Drawing</h1>
    <div>
      <canvas id="drawing-canvas" width="480" height="420"></canvas>
    </div>
    <div id="colorholder">
      <button id="eraser">消しゴム</button>
    </div>
  </div>
</template>

<script>
import { FireSQL } from 'firesql.js'

const createTable = `CREATE TABLE IF NOT EXISTS drawing (
  coords VARCHAR(45) NOT NULL,
  color VARCHAR(45) NULL,
  PRIMARY KEY (coords))`

function getURL(key, _default) {
  const url = new URL(document.location).searchParams.get(key)
  return url || _default
}

export default {
  mounted() {
    const db = new FireSQL(getURL('url', 'http://localhost:8080'))
    db.query(createTable)
    // Set up some globals
    const pixSize = 8
    let lastPoint = null
    let currentColor = '000'
    let mouseDown = 0

    // Create a reference to the pixel data for our drawing.
    // const pixelDataRef = new Firebase('https://INSTANCE.firebaseio.com/drawing')

    // Set up our canvas
    const myCanvas = document.getElementById('drawing-canvas')
    const myContext = myCanvas.getContext ? myCanvas.getContext('2d') : null
    if (myContext == null) {
      alert(
        'You must use a browser that supports HTML5 Canvas to run this demo.'
      )
      return
    }

    document
      .getElementById('eraser')
      .addEventListener('click', () => (currentColor = 'eraser'))
    // Setup each color palette & add it to the screen
    const colors = [
      'fff',
      '000',
      'f00',
      '0f0',
      '00f',
      '88f',
      'f8d',
      'f88',
      'f05',
      'f80',
      '0f8',
      'cf0',
      '08f',
      '408',
      'ff8',
      '8ff'
    ]
    for (const c in colors) {
      // eslint-disable-next-line no-undef
      const item = $('<div/>')
        .css('background-color', '#' + colors[c])
        .addClass('colorbox')
      item.click(
        (function() {
          const col = colors[c]
          return function() {
            currentColor = col
          }
        })()
      )
      item.appendTo('#colorholder')
    }

    // Keep track of if the mouse is up or down
    myCanvas.onmousedown = function() {
      mouseDown = 1
    }
    myCanvas.onmouseout = myCanvas.onmouseup = function() {
      mouseDown = 0
      lastPoint = null
    }

    // Draw a line from the mouse's last position to its current position
    const drawLineOnMouseMove = function(e) {
      if (!mouseDown) return

      // Bresenham's line algorithm. We use this to ensure smooth lines are drawn
      // eslint-disable-next-line no-undef
      const offset = $('canvas').offset()
      const x1 = Math.floor((e.pageX - offset.left) / pixSize - 1)
      const y1 = Math.floor((e.pageY - offset.top) / pixSize - 1)
      let x0 = lastPoint == null ? x1 : lastPoint[0]
      let y0 = lastPoint == null ? y1 : lastPoint[1]
      const dx = Math.abs(x1 - x0)
      const dy = Math.abs(y1 - y0)
      const sx = x0 < x1 ? 1 : -1
      const sy = y0 < y1 ? 1 : -1
      let err = dx - dy
      while (true) {
        if (currentColor === 'eraser') {
          clearPixel({ coords: `${x0}:${y0}` })
          db.query(`DELETE FROM drawing WHERE coords = '${x0}:${y0}'`)
        } else {
          drawPixel({ coords: `${x0}:${y0}`, color: currentColor })
          // write the pixel into Firebase, or if we are drawing white, remove the pixel
          // pixelDataRef
          //   .child(x0 + ':' + y0)
          //   .set(currentColor === 'fff' ? null : currentColor)
          db.query(
            `INSERT INTO drawing (coords, color) VALUES ('${x0}:${y0}', '${currentColor}') ON DUPLICATE KEY UPDATE color = '${currentColor}'`
          )
        }

        if (x0 === x1 && y0 === y1) break
        const e2 = 2 * err
        if (e2 > -dy) {
          err = err - dy
          x0 = x0 + sx
        }
        if (e2 < dx) {
          err = err + dx
          y0 = y0 + sy
        }
      }
      lastPoint = [x1, y1]
    }
    // eslint-disable-next-line no-undef
    $(myCanvas).mousemove(drawLineOnMouseMove)
    // eslint-disable-next-line no-undef
    $(myCanvas).mousedown(drawLineOnMouseMove)

    // Add callbacks that are fired any time the pixel data changes and adjusts the canvas appropriately.
    // Note that child_added events will be fired for initial pixel data as well.
    const drawPixel = function(row) {
      const coords = row.coords.split(':')
      myContext.fillStyle = '#' + row.color
      myContext.fillRect(
        parseInt(coords[0]) * pixSize,
        parseInt(coords[1]) * pixSize,
        pixSize,
        pixSize
      )
    }
    const clearPixel = function(row) {
      const coords = row.coords.split(':')
      myContext.clearRect(
        parseInt(coords[0]) * pixSize,
        parseInt(coords[1]) * pixSize,
        pixSize,
        pixSize
      )
    }

    // pixelDataRef.on('child_added', drawPixel)
    // pixelDataRef.on('child_changed', drawPixel)
    // pixelDataRef.on('child_removed', clearPixel)
    db.on('drawing', (rows, type) => {
      if (type === 'WriteRowsEvent') {
        for (const row of rows) {
          drawPixel(row.values)
        }
      } else if (type === 'UpdateRowsEvent') {
        for (const row of rows) {
          drawPixel(row.after_values)
        }
      } else if (type === 'DeleteRowsEvent') {
        for (const row of rows) {
          clearPixel(row.values)
        }
      }
    })
  }
}
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
}
.title {
  font-family: 'Quicksand', 'Source Sans Pro', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}

body {
  margin-top: 10px;
  margin-left: auto;
  margin-right: auto;
  width: 500px;
  background-color: #f8f8f8;
  font-size: 24px;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  color: #424547;
  text-align: center;
}

h1 {
  margin: 64px 0 32px;
  font-size: 36px;
  font-weight: bold;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  color: #424547;
  white-space: nowrap;
}

h3 {
  font-size: 24px;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  color: #424547;
}

p {
  font-size: 16px;
}

input {
  font-size: 24px;
}

input[type='text'] {
  color: #424547;
  border: 1px solid #c2c2c2;
  background-color: white;
}

em {
  font-style: normal;
  font-weight: bold;
  color: black;
}

/* Drawing */
#colorholder {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 480px;
  height: 33px;
  border: 2px solid #424547;
  margin-top: 5px;
  margin-left: auto;
  margin-right: auto;
}

#drawing-canvas {
  border: 3px solid #999;
}

.colorbox {
  width: 22px;
  height: 22px;
  margin: 1px;
  display: inline-block;
  border: 3px solid black;
}
</style>
