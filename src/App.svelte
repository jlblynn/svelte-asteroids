<script>

  import { onMount } from 'svelte';

  let canvas;
  let ctx;

  const FPS = 30;

  const GAME_LIVES = 3;
  const SHIP_SIZE = 30;
  const TURN_SPEED = 360;
  const FRICTION = 0.7;
  const SHIP_THRUST = 5;

  const SHOW_BOUNDING = false;

  const SHIP_EXPLODE_DURATION = 0.3;  
  const SHIP_INVISIBLE_DURATION = 3;
  const SHIP_BLINK_DURATION = 0.1;

  const LASER_MAX = 10;
  const LASER_SPEED = 300;
  const LASER_DIST = 0.5;
  const LASER_EXPLODE_DURATION = 0.1;

  const AST_NUM = 3;
  const AST_SIZE = 100;
  const AST_SPEED = 50;
  const AST_VERT = 10;
  const AST_JAG = 0.4;

  const TEXT_FADE_TIME = 2.5;
  const TEXT_SIZE = 80;

  let left = false;
  let right = false;
  let thrusting = false;
  let fire = false;

  let key;

  let score = 0;

  // setup game parameters
  let level, lives, ast, ship, text, textAlpha;

  onMount(() => {
		ctx = canvas.getContext('2d');

    ctx.fillRect(0, 0, canvas.width, canvas.height);


    newGame();

    // set up the game loop
    setInterval(update, 1000 / FPS);
  });

  function createAsteroidBelt() {
    ast = [];
    let x, y;
    for (var i = 0; i < AST_NUM + level; i++) {
      do {
        x = Math.floor(Math.random() * canvas.width);
        y = Math.floor(Math.random() * canvas.height);
      } while (distBetweenPoints(ship.x, ship.y, x, y) < AST_SIZE * 2 + ship.r);
      
      ast.push(newAsteroid(x, y, Math.ceil(AST_SIZE / 2)));
    }
  }

  function destroyAsteroid(index) {
    let x = ast[index].x;
    let y = ast[index].y;
    let r = ast[index].r;

    // split the asteroid in 2 if big enough
    if (r == Math.ceil(AST_SIZE / 2)) {
      ast.push(newAsteroid(x, y, Math.ceil(AST_SIZE / 4)));
      ast.push(newAsteroid(x, y, Math.ceil(AST_SIZE / 4)));
    } else if (r == Math.ceil(AST_SIZE / 4)) {
      ast.push(newAsteroid(x, y, Math.ceil(AST_SIZE / 8)));
      ast.push(newAsteroid(x, y, Math.ceil(AST_SIZE / 8)));
    }

    // destroy asteroid
    ast.splice(index, 1);

    // new level when no more asteroids
    if (ast.length == 0) {
      level++;
      newLevel();
    }
  }

  function distBetweenPoints(x1, y1, x2, y2) {
    return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
  }

  function drawShip(x, y, a, color = "black") {
    ctx.strokeStyle = color;
    ctx.lineWidth = SHIP_SIZE / 20;
    ctx.beginPath();
    ctx.moveTo(
      // bow of the ship
      x + 4 / 3 * ship.r * Math.cos(a),
      y - 4 / 3 * ship.r * Math.sin(a)
    );
    ctx.lineTo(
      // port stern of the ship
      x - ship.r * (2 / 3 * Math.cos(a) + Math.sin(a)),
      y + ship.r * (2 / 3 * Math.sin(a) - Math.cos(a))
    );
    ctx.lineTo(
      // starboard stern of the ship
      x - ship.r * (2 / 3 * Math.cos(a) - Math.sin(a)),
      y + ship.r * (2 / 3 * Math.sin(a) + Math.cos(a))
    );
    // draw last side of ship automatically
    ctx.closePath();
    ctx.stroke();
  }

  function explodeShip() {
    ship.explodeTime = Math.ceil(SHIP_EXPLODE_DURATION * FPS);
  }

  function gameOver() {
    ship.dead = true;
    text = "Game Over";
    textAlpha = 1.0;
    score == 0;
  }

  function newAsteroid(x, y, r) {
    let lvlMult = 1 + 0.1 * level;
    let asteroid = {
      x: x,
      y: y,
      xv: Math.random() * AST_SPEED * lvlMult / FPS * (Math.random() < 0.5 ? 1 : -1),
      yv: Math.random() * AST_SPEED * lvlMult / FPS * (Math.random() < 0.5 ? 1 : -1),
      r: r,
      a: Math.random() * Math.PI * 2, // in radians
      vert: Math.random() * (AST_VERT + 1) + AST_VERT / 2,
      offset: []
    }

    // create the vertex offset array
    for (var i = 0; i < asteroid.vert; i++) {
      asteroid.offset.push(Math.random() * AST_JAG * 2 + 1 - AST_JAG);
    }

    return asteroid;
  }

  function newShip() {
    return {
      x: canvas.width / 2,
      y: canvas.height / 2,
      r: SHIP_SIZE / 2,
      a: 90 / 180 * Math.PI, // convert to radians
      rot: 0,
      blinkNum: Math.ceil(SHIP_INVISIBLE_DURATION / SHIP_BLINK_DURATION),
      blinkTime: Math.ceil(SHIP_BLINK_DURATION * FPS),
      canShoot: true,
      dead: false,
      lasers: [],
      explodeTime: 0,
      thrusting: false,
      thrust: {
        x: 0,
        y: 0
      }
    }
  }

  function newGame() {
    level = 0;
    lives = GAME_LIVES;
    ship = newShip();
    newLevel();
  }

  function newLevel() {
    text = "Level: " + (level + 1);
    textAlpha = 1.0;
    createAsteroidBelt();

  }

  function shootLaser() {

    if (ship.canShoot && ship.lasers.length < LASER_MAX) {
      ship.lasers.push({
        x: ship.x + 4 / 3 * ship.r * Math.cos(ship.a),
        y: ship.y - 4 / 3 * ship.r * Math.sin(ship.a),
        xv: LASER_SPEED * Math.cos(ship.a) / FPS,
        yv: -LASER_SPEED * Math.sin(ship.a) / FPS,
        dist: 0,
        explodeTime: 0
      })
    }

    ship.canShoot = false;
  }

	function handleKeydown(event) {

    if (ship.dead) {
      return;
    }

		key = event.key;

    if (key == "ArrowLeft") {
      left = true;
      ship.rot = TURN_SPEED / 180 * Math.PI / FPS;
    }
    if (key == "ArrowRight") {
      right = true;
      ship.rot = -TURN_SPEED / 180 * Math.PI / FPS;
    }
    if (key == "ArrowUp") {
      thrusting = true;
    }
    if (key === ' ') {
      fire = true;
      shootLaser();
    }
	}

  function handleKeyup(event) {

    if (ship.dead) {
      return;
    }

		key = event.key;

    if (key == "ArrowLeft") {
      ship.rot = 0;
      left = false;
    }
    if (key == "ArrowRight") {
      ship.rot = 0;
      right = false;
    }
    if (key == "ArrowUp") {
      thrusting = false;
    }
    if (key === " ") {
      fire = false;
      ship.canShoot = true;
    }

	}

  function update() {
    let blinkOn = ship.blinkNum % 2 == 0;
    let exploding = ship.explodeTime > 0;

    // draw space
    ctx.fillStyle = "white";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // draw asteroids
    let x, y, r, a, vert, offset;
    for (let i = 0; i < ast.length; i++) {

      ctx.strokeStyle = "black";
      ctx.fillStyle = "slategrey";
      ctx.lineWidth = SHIP_SIZE / 20;

      // get asteroid properties
      x = ast[i].x;
      y = ast[i].y;
      r = ast[i].r;
      a = ast[i].a;
      vert = ast[i].vert;
      offset = ast[i].offset;

      // draw a path
      ctx.beginPath();
      ctx.moveTo(
        x + r * offset[0] * Math.cos(a),
        y + r * offset[0] * Math.sin(a)
      );

      // draw the polygon
      for (var j = 1; j < vert; j++) {
        ctx.lineTo(
          x + r * offset[j] * Math.cos(a + j * Math.PI * 2 / vert),
          y + r * offset[j] * Math.sin(a + j * Math.PI * 2 / vert)
        )
      }

      ctx.closePath();
      ctx.stroke();
      ctx.fill();

      if (SHOW_BOUNDING) {
        ctx.strokeStyle = "lime";
        ctx.beginPath();
        ctx.arc(x, y, r, 0, Math.PI * 2, false);
        ctx.stroke();
      }
    }

    // thrust the ship
    if(thrusting && !ship.dead) {
      ship.thrust.x += SHIP_THRUST * Math.cos(ship.a) / FPS;
      ship.thrust.y -= SHIP_THRUST * Math.sin(ship.a) / FPS;

      if (!exploding && blinkOn) {
        // animate flame
        ctx.fillStyle = "aqua";
        ctx.strokeStyle = "orange";
        ctx.lineWidth = SHIP_SIZE / 10;
        ctx.beginPath();
        ctx.moveTo(
          // rear left
          ship.x - ship.r * (2 / 3 * Math.cos(ship.a) + 0.5 * Math.sin(ship.a)),
          ship.y + ship.r * (2 / 3 * Math.sin(ship.a) - 0.5 * Math.cos(ship.a))
        );
        ctx.lineTo(
          // stern
          ship.x - ship.r * 5 / 3 * Math.cos(ship.a),
          ship.y + ship.r * 5 / 3 * Math.sin(ship.a)
        );
        ctx.lineTo(
          // rear right
          ship.x - ship.r * (2 / 3 * Math.cos(ship.a) - 0.5 * Math.sin(ship.a)),
          ship.y + ship.r * (2 / 3 * Math.sin(ship.a) + 0.5 * Math.cos(ship.a))
        );
        // draw last side of ship automatically
        ctx.closePath();
        ctx.fill();
        ctx.stroke();
      }
    } else {
      ship.thrust.x -= FRICTION * ship.thrust.x / FPS;
      ship.thrust.y -= FRICTION * ship.thrust.y / FPS;
    }

    // draw ship
    if (!exploding) {
      if (blinkOn && !ship.dead) {
       drawShip(ship.x, ship.y, ship.a)
      }

      // handle blinking
      if (ship.blinkNum > 0) {

        ship.blinkTime--;

        if (ship.blinkTime == 0) {
          ship.blinkTime = Math.ceil(SHIP_BLINK_DURATION * FPS);
          ship.blinkNum--;
        }
      }
    } else {
      // draw the explosion
      ctx.fillStyle = "darkred";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r * 1.7, 0, Math.PI * 2, false);
      ctx.fill();

      ctx.fillStyle = "red";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r * 1.4, 0, Math.PI * 2, false);
      ctx.fill();

      ctx.fillStyle = "orange";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r * 1.1, 0, Math.PI * 2, false);
      ctx.fill();

      ctx.fillStyle = "yellow";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r * 0.8, 0, Math.PI * 2, false);
      ctx.fill();

      ctx.fillStyle = "white";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r * 0.5, 0, Math.PI * 2, false);
      ctx.fill();
    }

    // show ship's collusion circle
    if (SHOW_BOUNDING) {
      ctx.strokeStyle = "lime";
      ctx.beginPath();
      ctx.arc(ship.x, ship.y, ship.r, 0, Math.PI * 2, false);
      ctx.stroke();
    }

    // draw lasers
    for (let i = 0; i < ship.lasers.length; i++) {
      if (ship.lasers[i].explodeTime == 0) {
        ctx.fillStyle = "lime";
        ctx.beginPath();
        ctx.arc(ship.lasers[i].x, ship.lasers[i].y, SHIP_SIZE / 15, 0, Math.PI * 2, false);
        ctx.fill();
      } else {
        // draw the explosion
        ctx.fillStyle = "orange";
        ctx.beginPath();
        ctx.arc(ship.lasers[i].x, ship.lasers[i].y, ship.r * 0.75, 0, Math.PI * 2, false);
        ctx.fill();
        ctx.fillStyle = "red";
        ctx.beginPath();
        ctx.arc(ship.lasers[i].x, ship.lasers[i].y, ship.r * 0.5, 0, Math.PI * 2, false);
        ctx.fill();
        ctx.fillStyle = "yellow";
        ctx.beginPath();
        ctx.arc(ship.lasers[i].x, ship.lasers[i].y, ship.r * 0.25, 0, Math.PI * 2, false);
        ctx.fill();
      }
    }

    // detect laser hits
    let ax, ay, ar, lx, ly;
    for (let i = ast.length - 1; i >= 0; i--) {

      // get asteroid properties
      ax = ast[i].x;
      ay = ast[i].y;
      ar = ast[i].r;

      // loop over the lasers
      for (let j = ship.lasers.length - 1; j >= 0; j--) {

        // get laser properties
        lx = ship.lasers[j].x;
        ly = ship.lasers[j].y;

        // detect hits
        if (ship.lasers[j].explodeTime == 0 && distBetweenPoints(ax, ay, lx, ly) < ar) {

          // destroy the asteroid and add laser explosion
          destroyAsteroid(i);
          ship.lasers[j].explodeTime = Math.ceil(LASER_EXPLODE_DURATION * FPS);
          score++;

          break;
        }
      }
    }

    // check for asteroid collisions
    if (!exploding) {
      if (ship.blinkNum == 0 && !ship.dead) {
        for (let i = 0; i < ast.length; i++) {
          if (distBetweenPoints(ship.x, ship.y, ast[i].x, ast[i].y) < ship.r + ast[i].r) {
            explodeShip();
            destroyAsteroid(i);
            break;
          }
        }
      }

      // rotate the ship
      ship.a += ship.rot;

      // move the ship
      ship.x += ship.thrust.x;
      ship.y += ship.thrust.y;
    } else {
      ship.explodeTime--;

      if (ship.explodeTime == 0) {
        lives--;
        if (lives == 0) {
          gameOver();
        } else {
          ship = newShip();
        }
      }
    }

    // handle edge of the screen
    if (ship.x < 0 - ship.r) {
      ship.x = canvas.width + ship.r;
    } else if (ship.x > canvas.width + ship.r) {
      ship.x = 0 - ship.r;
    }
    if (ship.y < 0 - ship.r) {
      ship.y = canvas.width + ship.r;
    } else if (ship.y > canvas.width + ship.r) {
      ship.y = 0 - ship.r;
    }

    // move the lasers
    for (let i = ship.lasers.length - 1; i >= 0; i--) {

      // check laser distance
      if (ship.lasers[i].dist > LASER_DIST * canvas.width) {
        ship.lasers.splice(i, 1);
        continue;
      }

      // handle the explosion
      if (ship.lasers[i].explodeTime > 0) {
        ship.lasers[i].explodeTime--;

        // destroy the laser after duration
        if (ship.lasers[i].explodeTime == 0) {
          ship.lasers.splice(i, 1);
          continue;
        }
      } else {
        // move the laser
        ship.lasers[i].x += ship.lasers[i].xv;
        ship.lasers[i].y += ship.lasers[i].yv;

        // calculate the distance travelled
        ship.lasers[i].dist += Math.sqrt(Math.pow(ship.lasers[i].xv, 2) + Math.pow(ship.lasers[i].yv, 2));
      }

      // handle edge of screen
      if (ship.lasers[i].x < 0) {
        ship.lasers[i].x = canvas.width;
      } else if (ship.lasers[i].x > canvas.width) {
        ship.lasers[i].x = 0;
      }
      if (ship.lasers[i].y < 0) {
        ship.lasers[i].y = canvas.height;
      } else if (ship.lasers[i].y > canvas.height) {
        ship.lasers[i].y = 0;
      }
    }

    // draw the game text
    if (textAlpha >= 0) {
      ctx.textAlign = "center";
      ctx.textBaseLine = "middle";
      ctx.fillStyle = "rgba(0, 0, 0, " + textAlpha + ")";
      ctx.font = "caps " + TEXT_SIZE + "px";
      ctx.fillText(text, canvas.width / 2, canvas.height * 0.75);
      textAlpha -= (1.0 / TEXT_FADE_TIME / FPS);
    } else if (ship.dead) {
      newGame();
    }

    // draw lives left
    let lifeColor;
    for (let i = 0; i < lives; i++) {
      lifeColor = exploding && i == lives -1 ? "red" : "black";
      drawShip(SHIP_SIZE + i * SHIP_SIZE * 1.2, SHIP_SIZE, 0.5 * Math.PI, lifeColor);
    }

    // move the asteroids
    for (let i = 0; i < ast.length; i++) {
      ast[i].x += ast[i].xv;
      ast[i].y += ast[i].yv;

      // handle edge of screen
      if (ast[i].x < 0 - ast[i].r) {
        ast[i].x = canvas.width + ast[i].r;
      } else if (ast[i].x > canvas.width + ast[i].r) {
        ast[i].x = 0 - ast[i].r
      }
      if (ast[i].y < 0 - ast[i].r) {
        ast[i].y = canvas.height + ast[i].r;
      } else if (ast[i].y > canvas.height + ast[i].r) {
        ast[i].y = 0 - ast[i].r
      }
    }

  }

  let leftButton, rightButton, forwardButton, shootButton;

  function turnLeft() {
    if (ship.dead) {
      return;
    }

    leftButton.ontouchstart = function () {
      left = true;
      ship.rot = TURN_SPEED / 180 * Math.PI / FPS;
    }
    leftButton.ontouchend = function () {
      left = false;
      ship.rot = 0;
    }
  }

  function turnRight() {
    if (ship.dead) {
      return;
    }

    rightButton.ontouchstart = function () {
      left = true;
      ship.rot = -TURN_SPEED / 180 * Math.PI / FPS;
    }
    rightButton.ontouchend = function () {
      left = false;
      ship.rot = 0;
    }
  }

  function moveForward() {
    if (ship.dead) {
      return;
    }
    forwardButton.ontouchstart = function () {
      thrusting = true;
    }
    forwardButton.ontouchend = function () {
      thrusting = false;
    }
  }

  function shoot() {
    if (ship.dead) {
      return;
    }
    shootButton.ontouchstart = function () {
      fire = true;
      shootLaser();
    }
    shootButton.ontouchend = function () {
      fire = false;
      ship.canShoot = true;
    }
  }




</script>

<svelte:head>
  <title> • &#128126; Asteroids </title>
</svelte:head>

<svelte:window on:keydown={handleKeydown} on:keyup={handleKeyup}/>

  <canvas 
    id="gameCanvas" 
    bind:this={canvas} 

    width={700}
    height= {500}
  >

  </canvas>

  <h2 class="score">Score: { score }</h2>

  <div class="mobile-buttons">
    
     <div id="forward-button" class="button" bind:this={forwardButton} on:click={moveForward}>
      <div class="button-outline">
        <p>forward</p>
      </div>
      <div class="button-shadow"></div>
    </div>

    <div id="left-button" class="button" bind:this={leftButton} on:click={turnLeft}>
      <div class="button-outline">
        <p>left</p>
      </div>
      <div class="button-shadow"></div>
    </div>

    <div id="right-button" class="button" bind:this={rightButton} on:click={turnRight}>
      <div class="button-outline">
        <p>right</p>
      </div>
      <div class="button-shadow"></div>
    </div>

    <div id="shoot-button" class="button" bind:this={shootButton} on:click={shoot}>
      <div class="button-outline">
        <p>fire!</p>
      </div>
      <div class="button-shadow"></div>
    </div>


  </div>



<style>

  @media (min-width: 320px) and (max-width: 480px) {

    #forward-button > div {
      margin-left: 60px;
    }

    #left-button > div {
      margin-top: 60px;
    }

    #left-button .button-shadow {
      margin-top: 70px;
      margin-left: 2px;
    }

    #right-button > div {
      margin-top: 60px;
      margin-left: 120px;
    }

    #right-button .button-shadow {
      margin-top: 70px;
      margin-left: 110px;
    }

    #shoot-button > div {
      margin-left: 240px;
    }

    #shoot-button .button-shadow {
      margin-left: 220px;
    }
    
    #gameCanvas {
      width: 350px;
      height: 250px;
    }

    .mobile-buttons {
      display: inline-block;
      padding: 20px;
    }

    .button-shadow {
      height: 40px;
      width: 100px;
      position: absolute;
      margin-top: 10px;
      border-radius: 10px;
      left: 10px;
      z-index: -1;
      background-color: transparent;
      border: 0.5px solid lightslategrey;
      -webkit-filter: blur(2px);
      -moz-filter: blur(2px);
      -o-filter: blur(2px);
      -ms-filter: blur(2px);
      filter: blur(2px);
    }

    .button-outline {
      height: 40px;
      width: 100px;
      border-radius: 10px;
      border: 1px solid black;
      background-color: transparent;
      position: absolute;
      z-index: 0;
    }

    .button {
      height: 40px;
      height: 100px;
      display: block;
      font-size: 12px;
      text-align: center;
      position: absolute;
    }
    
  }

  /* tablet */
  @media (min-width: 768px) and (max-width: 820px) {
    #forward-button > div {
      margin-left: 130px;
    }

    #left-button > div {
      margin-top: 60px;
      margin-left: 60px;
    }

    #left-button .button-shadow {
      margin-top: 70px;
      margin-left: 60px;
    }

    #right-button > div {
      margin-top: 60px;
      margin-left: 190px;
    }

    #right-button .button-shadow {
      margin-top: 70px;
      margin-left: 190px;
    }

    #shoot-button > div {
      margin-left: 540px;
    }

    #shoot-button .button-shadow {
      margin-left: 520px;
    }

    .mobile-buttons {
      display: inline-block;
      padding: 20px;
    }

    .button-shadow {
      height: 40px;
      width: 100px;
      position: absolute;
      margin-top: 10px;
      border-radius: 10px;
      left: 10px;
      z-index: -1;
      background-color: transparent;
      border: 0.5px solid lightslategrey;
      -webkit-filter: blur(2px);
      -moz-filter: blur(2px);
      -o-filter: blur(2px);
      -ms-filter: blur(2px);
      filter: blur(2px);
    }

    .button-outline {
      height: 40px;
      width: 100px;
      border-radius: 10px;
      border: 1px solid black;
      background-color: transparent;
      position: absolute;
      z-index: 0;
    }

    .button {
      height: 40px;
      height: 100px;
      display: block;
      font-size: 12px;
      text-align: center;
      position: absolute;
    }
  }

  /* ipad pro */
  @media (min-width: 821px) and (max-width: 1025px){
    #forward-button > div {
      margin-left: 200px;
    }

    #left-button > div {
      margin-top: 60px;
      margin-left: 140px;
    }

    #left-button .button-shadow {
      margin-top: 70px;
      margin-left: 140px;
    }

    #right-button > div {
      margin-top: 60px;
      margin-left: 270px;
    }

    #right-button .button-shadow {
      margin-top: 70px;
      margin-left: 270px;
    }

    #shoot-button > div {
      margin-left: 740px;
    }

    #shoot-button .button-shadow {
      margin-left: 720px;
    }

    .mobile-buttons {
      display: inline-block;
      padding: 20px;
    }

    .button-shadow {
      height: 40px;
      width: 100px;
      position: absolute;
      margin-top: 10px;
      border-radius: 10px;
      left: 10px;
      z-index: -1;
      background-color: transparent;
      border: 0.5px solid lightslategrey;
      -webkit-filter: blur(2px);
      -moz-filter: blur(2px);
      -o-filter: blur(2px);
      -ms-filter: blur(2px);
      filter: blur(2px);
    }

    .button-outline {
      height: 40px;
      width: 100px;
      border-radius: 10px;
      border: 1px solid black;
      background-color: transparent;
      position: absolute;
      z-index: 0;
    }

    .button {
      height: 40px;
      height: 100px;
      display: block;
      font-size: 12px;
      text-align: center;
      position: absolute;
    }
  }

  @media (min-width: 1026px) {
    .mobile-buttons {
      display: none;
    }
  }

  .score {
    text-align: center;
  }

  #gameCanvas {
    border: 1px solid black;
    margin: 0 auto;
    display: block;
    margin-top: 50px;
  }

</style>