<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Billiard</title>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css" integrity="sha512-dTfge/zgoMYpP7QbHy4gWMEGsbsdZeCXz7irItjcC3sPUFtf0kuFbDz/ixG7ArTxmDjLXDmezHubeNikyKGVyQ==" crossorigin="anonymous">

    <link rel="stylesheet" href="styles.css">
    <!-- external js libs -->
    <script type="text/javascript" src="lib/three.js/build/three.min.js"></script>
    <script type="text/javascript" src="lib/cannon.min.js"></script>
    <script type="text/javascript" src="lib/stats.min.js"></script>
    <script type="text/javascript" src="lib/OrbitControls.js"></script>

    <script type="text/javascript" src="js/CannonUtils.js"></script>

    <script type="text/javascript" src="js/Arch.js"></script>
    <script type="text/javascript" src="js/Gui.js"></script>
    <script type="text/javascript" src="js/Hole.js"></script>
    <script type="text/javascript" src="js/LongWall.js"></script>
    <script type="text/javascript" src="js/ShortWall.js"></script>
    <script type="text/javascript" src="js/Table.js"></script>
    <script type="text/javascript" src="js/Ball.js"></script>
    <script type="text/javascript" src="js/WhiteBall.js"></script>
    <script type="text/javascript" src="js/TableLight.js"></script>
    <script type="text/javascript" src="js/Game.js"></script>
    <script type="text/javascript" src="js/EightBallGame.js"></script>
    <script type="text/javascript" src="js/main.js"></script>
  </head>

  <body onload="onLoad()">
    <div id="overlay">
      <div id="loading">
        <h1>Loading, please wait...</h1>
        <div class="progress">
          <div id="prog-bar" class="progress-bar" role="progressbar" aria-valuenow="0"
            aria-valuemin="0" aria-valuemax="100">
            <span class="sr-only">Loading...</span>
          </div>
        </div>
      </div>

      <div id="mainMenu" class="hide">
        <div>
          <h1>8 Ball</h1>
          <button type="button" onclick="gui.play8BallClicked()" class="btn btn-primary btn-lg btn-block">Let's Go !</button>

        </div>
      </div>

      <div id="controlsHud" class="hide">
        <div class="player player1 unknown">
          <h5>Player 1 (<span></span>)</h5>

          <ul>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
          </ul>
        </div>

        <div class="player player2 unknown">
          <h5>Player 2 (<span></span>)</h5>

          <ul>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
            <li>?</li>
          </ul>
        </div>

        <div class="shootControls">
          <div class="power">
            Strength:
            <form>
              <input id="range_strength" type="range" name="strength" min="10" max="100">
            </form>
          </div>

          <button id="btn_ball" type="button" class="btn btn-success">Hit</button>
        </div>

        <ul class="gamelog list-unstyled">

        </ul>

        <div class="timer">
          30
        </div>
        <div id="fps_stats_container"></div>
      </div>

      <div id="gameover" class="hide">
        <h1>Player won!</h1>
        <h2>Thank you for playing.</h2>
      </div>

      <div id="canvas"></div>
    </div>

    <div class="jumbotron guide">
      <p>8 Ball project </p>
      <p>Yaser Bavaghar - Mohadeseh Pirkari - Fateme Fazlalian</p>
    </div>
    <div id="status"></div>
	</body>
</html>

