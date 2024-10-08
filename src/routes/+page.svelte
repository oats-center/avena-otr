<script>
  import { onMount } from 'svelte';
  import { useLocalStorage, getLocalImage } from '$lib/localStorage.svelte.js';
  import { Sensor } from '$lib/sensor.js'
  import { paintBackground, paintSensors } from '$lib/paintCanvases.js';
  import SensorControls from '$lib/SensorControls.svelte'
  import background from "$lib/images/background.png";
  import temp_sensor from "$lib/images/temp_sensor.png";
  import press_sensor from "$lib/images/press_sensor.png";

  let sensors = useLocalStorage('sensors', []).value;
  let sensorColors = ['red', 'orange', 'yellow', 'green', 'blue', 'purple', 'grey', 'black'];
  let sensorGroups = ['', 'temperature', 'pressure']
  let currSensor = $state(null);
  let queuedSensor = $state(null);
  let bgCanvas = $state(null);
  let fgCanvas = $state(null);
  let bgContext, fgContext;
  let mouseX = 0;
  let mouseY = 0;
  let xMax = $state(0);
  let yMax = $state(0);
  let resizeTimeout;
  let animationFrameId;
  let backgroundImage = $state(null);
  let sensorImages = $state([]);

  onMount(() => {
    let tempBg = getLocalImage('background', "").value;
    const tempImage = new Image();
    tempImage.src = temp_sensor;
    tempImage.onload = function() {
      sensorImages.push(tempImage);
    }
    const pressImage = new Image();
    pressImage.src = press_sensor;
    pressImage.onload = function() { 
      sensorImages.push(pressImage);
    }
    let fgImgs = [tempImage, pressImage];
    const img = new Image();
    img.src = tempBg ? tempBg : background;
    img.onload = function () {
      backgroundImage = img;
      setupCanvases(fgImgs, img);
    }
//
    
    // Initial render
    window.addEventListener('resize', function(e) {
      resizeCanvas();
    })
  });
  
  function animateForeground() {
    if (currSensor) {
      renderForeground();
      // Keep requesting animation frames as long as currSensor is not null
      animationFrameId = requestAnimationFrame(animateForeground);
    } else {
      // Stop the animation when currSensor is null
      cancelAnimationFrame(animationFrameId);
      renderForeground();
    }
  }

  function setupCanvases(fgImage, bgImage) {
    xMax = window.innerWidth * 0.60;
    yMax = window.innerHeight * 0.80;
    
    let width = bgImage ? bgImage.width : backgroundImage.width;
    let height = bgImage ? bgImage.height : backgroundImage.height;


    let maxTopHeight = yMax * 0.75;
    let width_ratio = xMax / width;
    let height_ratio = maxTopHeight / height;

    if(width_ratio < height_ratio){
      xMax = width * width_ratio;
      yMax = height * width_ratio;
    } else {
      xMax = width * height_ratio;
      yMax = height * height_ratio;
    }

    bgContext = bgCanvas.getContext('2d');
    bgCanvas.width = xMax;
    bgCanvas.height = yMax;

    fgContext = fgCanvas.getContext('2d');
    fgCanvas.width = xMax;
    fgCanvas.height = yMax;

    renderBackground(bgImage);
    renderForeground(fgImage);
  }

  function renderBackground(img) {
    if(img) paintBackground(bgContext, img);
    else paintBackground(bgContext, backgroundImage);
  }

  function renderForeground(img) {
    if(img) paintSensors(fgContext, sensors, currSensor, img);
    else paintSensors(fgContext, sensors, currSensor, sensorImages);
  }

  function resizeCanvas() {
    clearTimeout(resizeTimeout);
    resizeTimeout = setTimeout(() => {
      setupCanvases();
    }, 200);
  }
  
  function handleMouseMove(event) {
    const rect = fgCanvas.getBoundingClientRect();
    mouseX = event.clientX - rect.left;
    mouseY = event.clientY - rect.top;
  }

  function generateRandomId() {
    const length = 5;
    const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    let result = '';
    for (let i = 0; i < length; i++) {
      result += characters.charAt(Math.floor(Math.random() * characters.length));
    }
    return result;
  }

  function addSensor() {
    sensors.push(new Sensor(generateRandomId(), 0, 0, 'red', 1, 'New Sensor', ""));
    currSensor = JSON.parse(JSON.stringify(sensors[sensors.length - 1]));
    animateForeground();
  }
  
  function updateSensors() {
    const index = sensors.findIndex(sensor => sensor.id === currSensor.id);
    if (index !== -1) {
      sensors[index] = new Sensor(
        currSensor.id,
        currSensor.x_pos,
        currSensor.y_pos,
        currSensor.color,
        currSensor.layer,
        currSensor.name,
        currSensor.group
      );
    }
    currSensor = null;
    renderForeground();
  }

  function deleteSensor() {
    const index = sensors.findIndex(sensor => sensor.id === currSensor.id);
    if (index !== -1) {
      sensors.splice(index, 1);
      renderForeground();
    }
    currSensor = null;
  }

  function handleClick() {
    const clickedSensor = sensors.find(sensor =>
      (sensor.group === "" && mouseX > sensor.x_pos * xMax - 5 && mouseX < sensor.x_pos * xMax + 5  &&
       mouseY > sensor.y_pos * yMax - 5 && mouseY < sensor.y_pos * yMax + 5) ||
       (sensor.group !== "" && mouseX > sensor.x_pos * xMax - 25 && mouseX < sensor.x_pos * xMax + 25  &&
       mouseY > sensor.y_pos * yMax - 25 && mouseY < sensor.y_pos * yMax + 25)
    ) || null;
    if (clickedSensor) {
      if (currSensor) {
        queuedSensor = { ...clickedSensor };
        cancel_modal.showModal();
      } else {
        queuedSensor = null;
        currSensor = { ...clickedSensor };
      }
      if(currSensor){
        animateForeground();
      }
    }
  }
</script>

<div class="flex">
  <div class="canvas_container">
    <canvas class="background" bind:this={bgCanvas}></canvas>
    <canvas class="foreground" onclick={handleClick} onmousemove={handleMouseMove} bind:this={fgCanvas}></canvas>
  </div>
  <SensorControls
    {currSensor}
    {sensorColors}
    {sensorGroups}
    onAddSensor={addSensor}
    onSaveSensor={updateSensors}
    onDeleteSensor={deleteSensor}
  />
</div>

<dialog id="cancel_modal" class='modal'>
  <div class="modal-box">
    <h3 class="text-lg font-bold">Cancel Changes?</h3>
    <h6>Pressing 'Yes' will delete all changes made without saving</h6>
    <div class="mt-5 flex">
      <form method="dialog">
        <button class="btn btn-primary">No</button>
        <button class="btn btn-error ml-5" onclick={ () => {
          queuedSensor ? currSensor = queuedSensor : currSensor = null;
          queuedSensor = null;
        }}>Yes</button>
      </form>
    </div>
  </div>
  <form method="dialog" class="modal-backdrop">
    <button>close</button>
  </form>
</dialog>

<dialog id="delete_modal" class='modal'>
  <div class="modal-box">
    <h3 class="text-lg font-bold">Delete Sensor?</h3>
    <h6>Pressing 'Yes' will delete the currently selected sensor and that data will be unrecoverable</h6>
    <div class="mt-5 flex">
      <form method="dialog">
        <button class="btn btn-primary">No</button>
        <button class="btn btn-error ml-5" onclick={ () => deleteSensor() }>Yes</button>
      </form>
    </div>
  </div>
  <form method="dialog" class="modal-backdrop">
    <button>close</button>
  </form>
</dialog>

<dialog id="save_modal" class='modal'>
  <div class="modal-box">
    <h3 class="text-lg font-bold">Save Changes?</h3>
    <h6>Pressing 'Yes' will override the already saved data with the new data.</h6>
    <div class="mt-5 flex">
      <form method="dialog">
        <button class="btn btn-primary">No</button>
        <button class="btn btn-success ml-5" onclick={ () => updateSensors() }>Yes</button>
      </form>
    </div>
  </div>
  <form method="dialog" class="modal-backdrop">
    <button>close</button>
  </form>
</dialog>


<style>
  .canvas_container {
    position: relative;
    width: 75%;
    height: 100%;
  }
  .foreground {
    z-index: 1
  }
  .background {
    z-index: 0;
  }
  canvas {
    position: absolute;
    top: 1;
    left: 10vw;
  }
</style>
