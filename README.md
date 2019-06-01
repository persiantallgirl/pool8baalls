//تخته بازی
var Arch = function (params) {
  this.body = new CANNON.Body({
    mass: 0, // جرم == 0 بدن را ثابت می کند
    material: Table.floorContactMaterial
  });

  params = params || {};
  //مقادیر پیش فرض
  this.position = params.position || { x: 0, y: 0, z: 0};

  //شعاع قوس نسبی، (شعاع واقعی از قوس کمی بزرگتر است ..)
  this.radius = params.radius || Ball.RADIUS + 2;

  this.box_autowidth = params.box_autowidth || false;
  this.box_width = params.box_width || 2;
  this.box_height = params.box_height || 5;
  this.box_thickness = params.box_thickness || 2;
  this.no_of_boxes = params.no_of_boxes || 5;

  this.body.position.set(this.position.x, this.position.y, this.position.z);
  var y_axis = new CANNON.Vec3(0, 1, 0);
  this.body.quaternion.setFromAxisAngle(y_axis, Math.PI);

  var box_increment_angle = Math.PI / (2 * this.no_of_boxes); //مقدار  پی یه برای زاویه مرکز جعبه به مرکز دایره
  // دریافت جعبه X-len با توجه به شعاع به طوری  همپوشانی وجود دارد
  var x_len = this.radius * Math.tan(box_increment_angle);

  if (!this.box_autowidth) {
    x_len = this.box_width;
  }

  // از فرم جعبه به عنوان شکل فرزند استفاده کنید
  var shape = new CANNON.Box(new CANNON.Vec3(x_len, this.box_height, this.box_thickness));

  for (var i = 0; i < this.no_of_boxes; i++) {
    var angle = box_increment_angle + (i * Math.PI / this.no_of_boxes);
    var b_x = Math.cos(angle);
    var b_z = Math.sin(angle);

    b_x *= this.radius + this.box_thickness;
    b_z *= this.radius + this.box_thickness;

    this.body.addShape(shape,
      new CANNON.Vec3(b_x, 0, b_z),
      createQuaternionFromAxisAngle(y_axis, Math.PI / 2 - angle));
  }
};

//توپ ها
var Ball = function (x, y, z, name, color) {
  this.color = typeof color === 'undefined' ? 0xcc0000 : color; //رنگ پیش فرض
  this.texture = 'images/balls/' + name + '.png';

  this.mesh = this.createMesh(x,y,z);
  this.sphere = new THREE.Sphere(this.mesh.position, Ball.RADIUS); //برای تشخیص تقاطع خط استفاده می شود
  scene.add(this.mesh);

  this.rigidBody = this.createBody(x,y,z);
  world.addBody(this.rigidBody);
  this.name = name;
  this.fallen = false;
};

Ball.RADIUS = 5.715 / 2; // cm
Ball.MASS = 0.170; // kg
Ball.contactMaterial = new CANNON.Material("ballMaterial");

/** بارگذاری نقشه ENV برای توپ.
  TODO: find a nicer place to do this. */
Ball.envMapUrls = [
  'images/skybox1/px.png', // positive x
  'images/skybox1/nx.png', // negative x
  'images/skybox1/py.png', // positive y
  'images/skybox1/ny.png', // negative y
  'images/skybox1/pz.png', // positive z
  'images/skybox1/nz.png'  // negative z
];
//بارگذاری بافت مکعب
var cubeTextureLoader = new THREE.CubeTextureLoader();
Ball.envMap = cubeTextureLoader.load(Ball.envMapUrls, function (tex) {
  Ball.envMap = tex;
});

Ball.prototype.onEnterHole = function () {
  this.rigidBody.velocity = new CANNON.Vec3(0);
  this.rigidBody.angularVelocity = new CANNON.Vec3(0);
  world.removeBody(this.rigidBody);
  eightballgame.coloredBallEnteredHole(this.name);
};

Ball.prototype.createBody = function (x,y,z) {
  var sphereBody = new CANNON.Body({
    mass: Ball.MASS, // kg
    position: new CANNON.Vec3(x,y,z), // m
    shape: new CANNON.Sphere(Ball.RADIUS),
    material: Ball.contactMaterial
  });

  sphereBody.linearDamping = sphereBody.angularDamping = 0.5; // کد سخت
  sphereBody.allowSleep = true;

  // Sleep parameters
  sphereBody.sleepSpeedLimit = 0.5; // توپ در جای خود بایستد خواهد کرد اگر سرعت <0.05 (سرعت == سرعت نرمال)
  sphereBody.sleepTimeLimit = 0.1; // تو\ پس از 1 ثانیه از می ایستد

  return sphereBody;
};
//ايجاد كردن
// مش
Ball.prototype.createMesh = function (x,y,z) {
  var geometry = new THREE.SphereGeometry(Ball.RADIUS, 16, 16);
  var material = new THREE.MeshPhongMaterial({
    specular: 0xffffff,
    shininess: 140,
    reflectivity: 0.1,
    envMap: Ball.envMap,
    combine: THREE.AddOperation,
    shading: THREE.SmoothShading
  });

  if (typeof this.texture === 'undefined') {
    material.color = new THREE.Color(this.color);
  } else {
    textureLoader.load(this.texture, function (tex) {
      material.map = tex;
      material.needsUpdate = true;
    });
  }

  var sphere = new THREE.Mesh(geometry, material);

  sphere.position.set(x,y,z);

  sphere.castShadow = true;
  sphere.receiveShadow = true;

  return sphere;
};

Ball.prototype.tick = function (dt) {
  this.mesh.position.copy(this.rigidBody.position);
  this.mesh.quaternion.copy(this.rigidBody.quaternion);

  // آیا توپ به یک سوراخ افتاده است؟
  if (this.rigidBody.position.y < -4 * Ball.RADIUS && !this.fallen) {
    this.fallen = true;
    this.onEnterHole();
  }
};

var wireframeMaterial;

var createQuaternionFromAxisAngle = function (axis, angle) {
  var q = new CANNON.Quaternion();
  q.setFromAxisAngle(axis,angle);
  return q;
};

/*
  Adapted from cannon.demo.js:
  Helper to draw wireframes of collision bodies
*/
var addCannonVisual = function (body, _color) {
  var color = typeof _color === 'undefined' ? 0xffffff : _color; //default color
  wireframeMaterial = new THREE.MeshBasicMaterial({color: color, wireframe: true});
  var s = 5;
  // What geometry should be used?
  var mesh;
  if (body instanceof CANNON.Body) {
    mesh = Cannonshape2mesh(body);
  }

  if (mesh) {
    //mesh.useQuaternion = true;
    scene.add(mesh);
  }
};
/** Adapted from cannon.demo.js:*/
Cannonshape2mesh = function (body) {
  var obj = new THREE.Object3D();

  for (var l = 0; l < body.shapes.length; l++) {
    var shape = body.shapes[l];
    var mesh;

    switch (shape.type) {
      case CANNON.Shape.types.SPHERE:
        var sphere_geometry = new THREE.SphereGeometry(shape.radius, 8, 8);
        mesh = new THREE.Mesh(sphere_geometry, wireframeMaterial);
        break;

      case CANNON.Shape.types.PARTICLE:
        mesh = new THREE.Mesh(this.particleGeo, this.particleMaterial);
        var s = this.settings;
        mesh.scale.set(s.particleSize, s.particleSize, s.particleSize);
        break;

      case CANNON.Shape.types.PLANE:
        var geometry = new THREE.PlaneGeometry(10, 10, 4, 4);
        mesh = new THREE.Object3D();
        var submesh = new THREE.Object3D();
        var ground = new THREE.Mesh(geometry, wireframeMaterial);
        ground.scale.set(100, 100, 100);
        submesh.add(ground);

        ground.castShadow = true;
        ground.receiveShadow = true;

        mesh.add(submesh);
        break;

      case CANNON.Shape.types.BOX:
        var box_geometry = new THREE.BoxGeometry(
          shape.halfExtents.x * 2,
          shape.halfExtents.y * 2,
          shape.halfExtents.z * 2
        );
        mesh = new THREE.Mesh(box_geometry, wireframeMaterial);
        break;

      case CANNON.Shape.types.CONVEXPOLYHEDRON:
        var geo = new THREE.Geometry();

        // Add vertices
        for (var i = 0; i < shape.vertices.length; i++) {
          var v = shape.vertices[i];
          geo.vertices.push(new THREE.Vector3(v.x, v.y, v.z));
        }

        for (var i = 0; i < shape.faces.length; i++) {
          var face = shape.faces[i];

          // add triangles
          var a = face[0];
          for (var j = 1; j < face.length - 1; j++) {
            var b = face[j];
            var c = face[j + 1];
            geo.faces.push(new THREE.Face3(a, b, c));
          }
        }

        geo.computeBoundingSphere();
        geo.computeFaceNormals();
        mesh = new THREE.Mesh(geo, wireframeMaterial);
        break;

      case CANNON.Shape.types.HEIGHTFIELD:
        var geometry = new THREE.Geometry();

        var v0 = new CANNON.Vec3();
        var v1 = new CANNON.Vec3();
        var v2 = new CANNON.Vec3();
        for (var xi = 0; xi < shape.data.length - 1; xi++) {
          for (var yi = 0; yi < shape.data[xi].length - 1; yi++) {
            for (var k = 0; k < 2; k++) {
              shape.getConvexTrianglePillar(xi, yi, k === 0);
              v0.copy(shape.pillarConvex.vertices[0]);
              v1.copy(shape.pillarConvex.vertices[1]);
              v2.copy(shape.pillarConvex.vertices[2]);
              v0.vadd(shape.pillarOffset, v0);
              v1.vadd(shape.pillarOffset, v1);
              v2.vadd(shape.pillarOffset, v2);
              geometry.vertices.push(
                new THREE.Vector3(v0.x, v0.y, v0.z),
                new THREE.Vector3(v1.x, v1.y, v1.z),
                new THREE.Vector3(v2.x, v2.y, v2.z)
              );
              var i = geometry.vertices.length - 3;
              geometry.faces.push(new THREE.Face3(i, i + 1, i + 2));
            }
          }
        }
        geometry.computeBoundingSphere();
        geometry.computeFaceNormals();
        mesh = new THREE.Mesh(geometry, wireframeMaterial);
        break;

      case CANNON.Shape.types.TRIMESH:
        var geometry = new THREE.Geometry();

        var v0 = new CANNON.Vec3();
        var v1 = new CANNON.Vec3();
        var v2 = new CANNON.Vec3();
        for (var i = 0; i < shape.indices.length / 3; i++) {
          shape.getTriangleVertices(i, v0, v1, v2);
          geometry.vertices.push(
              new THREE.Vector3(v0.x, v0.y, v0.z),
              new THREE.Vector3(v1.x, v1.y, v1.z),
              new THREE.Vector3(v2.x, v2.y, v2.z)
          );
          var j = geometry.vertices.length - 3;
          geometry.faces.push(new THREE.Face3(j, j + 1, j + 2));
        }
        geometry.computeBoundingSphere();
        geometry.computeFaceNormals();
        mesh = new THREE.Mesh(geometry, wireframeMaterial);
        break;

      default:
        throw "Visual type not recognized: " + shape.type;
    }

    mesh.receiveShadow = false;
    mesh.castShadow = false;

    var o = body.shapeOffsets[l];
    var q = body.shapeOrientations[l];
    mesh.position.set(o.x, o.y, o.z);
    mesh.quaternion.set(q.x, q.y, q.z, q.w);

    obj.add(mesh);
  }

  obj.position.copy(body.position);
  obj.quaternion.copy(body.quaternion);

  return obj;
};

//هشت توپ بازی
var EightBallGame = function()
{
  this.numbered_balls_on_table = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15];
  this.turn = 'player1';
  this.sides = {
    'player1': '?',
    'player2': '?'
  };
  //کدجابجایی رخ داده است

  this.pocketingOccurred = false;

  this.state = 'notstarted';

  this.ticker = undefined;

  gui.setupGameHud();
  setTimeout(this.startTurn, 2000);
}

EightBallGame.prototype.startTurn = function () {
  if (eightballgame.state == 'gameover') {
    return;
  }
  // فعال کردن جنبش
  eightballgame.timer = 30;
  eightballgame.state = 'turn';
  gui.updateTurn(eightballgame.turn);
  gui.updateBalls(eightballgame.numbered_balls_on_table, eightballgame.sides.player1, eightballgame.sides.player2);

  eightballgame.tickTimer();
}

EightBallGame.prototype.whiteBallEnteredHole = function () {
  gui.log("White ball pocketed by " + eightballgame.turn + "!");
}

EightBallGame.prototype.coloredBallEnteredHole = function (name) {
  if (typeof name === 'undefined') return;
  var ballno = 0;
  for (var i = 0; i < eightballgame.numbered_balls_on_table.length; i++) {
    if (name == eightballgame.numbered_balls_on_table[i] + 'ball') {
      ballno = eightballgame.numbered_balls_on_table[i];
      eightballgame.numbered_balls_on_table.splice(i, 1);
      break;
    }
  }
  if (ballno == 0) {
    return;
  }

  if (ballno == 8) {
    if (eightballgame.numbered_balls_on_table.length > 1) {
      gui.log("Game over! 8 ball pocketed too early by " + this.turn);
      eightballgame.turn = eightballgame.turn == 'player1' ? 'player2': 'player1';
    }

    eightballgame.pocketingOccurred = true;

    // پیروزی!
    eightballgame.endGame();
  } else {
    if (eightballgame.sides.player1 == '?' || eightballgame.sides.player2 == '?') {
      eightballgame.sides[eightballgame.turn] = ballno < 8 ? 'solid' : 'striped';
      eightballgame.sides[eightballgame.turn == 'player1' ? 'player2' : 'player1'] = ballno > 8 ? 'solid' : 'striped';
      eightballgame.pocketingOccurred = true;
    } else {
      if ((eightballgame.sides[eightballgame.turn] == 'solid' && ballno < 8) || (eightballgame.sides[eightballgame.turn] == 'striped' && ballno > 8)) {
        // نوبت دیگر
        eightballgame.pocketingOccurred = true;
      } else {
        eightballgame.pocketingOccurred = false;
        gui.log(eightballgame.turn + " pocketed opponent's ball!");
      }
    }
  }
}
//خارج از زمان
EightBallGame.prototype.tickTimer = function () {
  gui.UpdateTimer(eightballgame.timer);
  if (eightballgame.timer == 0) {
    gui.log(eightballgame.turn + " ran out of time");
    eightballgame.state = "outoftime";
    eightballgame.switchSides();
  } else {
    eightballgame.timer--;
    eightballgame.ticker = setTimeout(eightballgame.tickTimer, 1000);
  }
}

EightBallGame.prototype.switchSides = function () {
  eightballgame.turn = eightballgame.turn == 'player1' ? 'player2': 'player1';

  setTimeout(eightballgame.startTurn, 1000);
}
//بازی تمام شد
EightBallGame.prototype.endGame = function () {
  eightballgame.state = 'gameover';
  var winner = eightballgame.turn == 'player1' ? 'Player 1' : 'Player 2';
  clearTimeout(eightballgame.ticker);
  gui.showEndGame(winner);
}
//در انتظار
EightBallGame.prototype.hitButtonClicked = function (strength) {
  if (game.balls[0].rigidBody.sleepState == CANNON.Body.SLEEPING && eightballgame.state == 'turn') {
    game.ballHit(strength);
    clearTimeout(eightballgame.ticker);
    eightballgame.state = 'turnwaiting';
    var x = setInterval(function() {
      if (game.balls[0].rigidBody.sleepState != CANNON.Body.SLEEPING) return;
      for (var i=1;i<game.balls.length;i++) {
        if (game.balls[i].rigidBody.sleepState != CANNON.Body.SLEEPING && eightballgame.numbered_balls_on_table.indexOf(Number(game.balls[i].name.split('ball')[0])) > -1) {
          return;
        }
      }

      if (eightballgame.pocketingOccurred) {
        setTimeout(eightballgame.startTurn, 1000);
      } else {
        eightballgame.switchSides();
      }

      eightballgame.pocketingOccurred = false;

      clearInterval(x);
    }, 30);
  }
}

//بازی
// تابع سازنده
var Game = function () {
  this.table = new Table();
  //TODO write a nice thing for automatically positioning the balls instead of this hardcoded crap?
  var X_offset = Table.LEN_X / 4;
  var X_offset_2 = 1.72; // this controls how tightly the balls are packed together on the x-axis

  this.balls = [
    new WhiteBall(),

    // ردیف اول
    new Ball(X_offset, Ball.RADIUS, 4 * Ball.RADIUS, '4ball'),
    new Ball(X_offset, Ball.RADIUS, 2 * Ball.RADIUS, '3ball'),
    new Ball(X_offset, Ball.RADIUS, 0, '14ball'),
    new Ball(X_offset, Ball.RADIUS, -2 * Ball.RADIUS, '2ball'),
    new Ball(X_offset, Ball.RADIUS, -4 * Ball.RADIUS, '15ball'),

    // ردیف دوم
    new Ball(X_offset - X_offset_2 * Ball.RADIUS, Ball.RADIUS, 3 * Ball.RADIUS, '13ball'),
    new Ball(X_offset - X_offset_2 * Ball.RADIUS, Ball.RADIUS, Ball.RADIUS, '7ball'),
    new Ball(X_offset - X_offset_2 * Ball.RADIUS, Ball.RADIUS, -1 * Ball.RADIUS, '12ball'),
    new Ball(X_offset - X_offset_2 * Ball.RADIUS, Ball.RADIUS, -3 * Ball.RADIUS, '5ball'),

    //ردیف سوم
    new Ball(X_offset - X_offset_2 * 2 * Ball.RADIUS, Ball.RADIUS, 2 * Ball.RADIUS, '6ball'),
    new Ball(X_offset - X_offset_2 * 2 * Ball.RADIUS, Ball.RADIUS, 0, '8ball'),
    new Ball(X_offset - X_offset_2 * 2 * Ball.RADIUS, Ball.RADIUS, -2 * Ball.RADIUS, '9ball'),

    //ردیف چهارم
    new Ball(X_offset - X_offset_2 * 3 * Ball.RADIUS, Ball.RADIUS, Ball.RADIUS, '10ball'),
    new Ball(X_offset - X_offset_2 * 3 * Ball.RADIUS, Ball.RADIUS, -1 * Ball.RADIUS, '11ball'),

    //ردیف \نجم
    new Ball(X_offset - X_offset_2 * 4 * Ball.RADIUS, Ball.RADIUS, 0, '1ball')
  ];
}

Game.prototype.tick = function (dt) {
  for (var i in this.balls) {
    this.balls[i].tick(dt);
  }
};

/** توپ را با قدرت داده شده به توپ برسانید. این
  توپ را به سمت آن حرکت می کند و به جلو حرکت می کند
  با موقعیت دوربین تعیین می شود / زاویه */
Game.prototype.ballHit = function (strength) {
  if (this.balls[0].rigidBody.sleepState == CANNON.Body.SLEEPING) {
    this.balls[0].hitForward(strength);
  }
};

//بازی توپ
var GameGui = function () {
  var btn_ball = document.getElementById('btn_ball');
  btn_ball.onclick = function () {
    eightballgame.hitButtonClicked(Number(document.getElementById('range_strength').value));
  };
  if (debug) document.getElementById('fps_stats_container').appendChild( stats.domElement );
};
GameGui.prototype.setupGameHud = function() {
  this.hide(document.getElementById('mainMenu'));
  this.show(document.getElementById('controlsHud'));
};
//اضافه کردن کلاس
GameGui.addClass = function (el, className) {
  if (el.classList) {
    el.classList.add(className);
  } else {
    el.className += ' ' + className;
  }
};
//حذف کلاس
GameGui.removeClass = function (el, className) {
  if (el.classList) {
    el.classList.remove(className);
  } else {
    el.className = el.className.replace(new RegExp('(^|\\b)' + className.split(' ').join('|') + '(\\b|$)', 'gi'), ' ');
  }
};
//نمونه اولیه
//اجرا
GameGui.prototype.show = function (node) {
  GameGui.removeClass(node, 'hide');
};

GameGui.prototype.hide = function (node) {
  GameGui.addClass(node, 'hide');
};

GameGui.prototype.play8BallClicked = function () {
  eightballgame = new EightBallGame();
};

GameGui.prototype.UpdateTimer = function(timerVal) {
  document.getElementsByClassName('timer')[0].textContent = timerVal;
};

GameGui.prototype.log = function(str) {
  var node = document.createElement('li');
  node.textContent = str;
  document.getElementsByClassName('gamelog')[0].appendChild(node);
};

GameGui.prototype.updateTurn = function(str) {
  GameGui.removeClass(document.getElementsByClassName('player1')[0], 'active');
  GameGui.removeClass(document.getElementsByClassName('player2')[0], 'active');
  GameGui.addClass(document.getElementsByClassName(str)[0], 'active');
};

GameGui.prototype.updateBalls = function(ballArr, p1side, p2side) {
  p1side = p1side == '?' ? 'unknown' : p1side;
  p2side = p2side == '?' ? 'unknown' : p2side;

  GameGui.removeClass(document.getElementsByClassName('player1')[0], 'solid');//جامد
  GameGui.removeClass(document.getElementsByClassName('player2')[0], 'solid');
  GameGui.removeClass(document.getElementsByClassName('player1')[0], 'striped');//راه راه
  GameGui.removeClass(document.getElementsByClassName('player2')[0], 'striped');
  GameGui.removeClass(document.getElementsByClassName('player1')[0], 'unknown');
  GameGui.removeClass(document.getElementsByClassName('player2')[0], 'unknown');
  GameGui.addClass(document.getElementsByClassName('player1')[0], p1side);
  GameGui.addClass(document.getElementsByClassName('player2')[0], p2side);

  if (p1side == 'unknown') {
    return;
  }
//ايجاد كردن
// عنصر
  var elem = document.createElement('ul');
  for (var i=1;i<8;i++) {
    var el = document.createElement('li');
    el.textContent = i;
    if (ballArr.indexOf(i) > -1) {

    } else {
      GameGui.addClass(el, 'pocketed');
    }

    elem.appendChild(el);
  }
  document.getElementsByClassName(p1side == 'solid' ? 'player1' : 'player2')[0].replaceChild(elem, document.getElementsByClassName(p1side == 'solid' ? 'player1' : 'player2')[0].children[1]);
  elem = document.createElement('ul');
  for (var i=9;i<16;i++) {
    var el = document.createElement('li');
    el.textContent = i;
    if (ballArr.indexOf(i) > -1) {

    } else {
      GameGui.addClass(el, 'pocketed');
    }

    elem.appendChild(el);
  }
  document.getElementsByClassName(p1side == 'striped' ? 'player1' : 'player2')[0].replaceChild(elem, document.getElementsByClassName(p1side == 'striped' ? 'player1' : 'player2')[0].children[1]);
};
//پایان بازی
GameGui.prototype.showEndGame = function(str) {
  document.getElementById("gameover").children[0].textContent = str + " won!";
  this.show(document.getElementById('gameover'));
}

//کیسه بیلیارد
var Hole = function (x, y, z, rotation) {
  //z = z -4.8;
  // تخته "دیوار"
  this.arch1 = new Arch({
    position: {x: x, y: y, z: z},
    no_of_boxes: 6,
    box_height: 6,
    box_autowidth: true,
    box_thickness: 0.1
  });
  // تخته "کف"
  this.arch2 = new Arch({
    position: {x: x, y: y - 1, z: z},
    no_of_boxes: 6,
    box_height: 1,
    box_width: 2.5,
    box_thickness: 3
  });
//تنظیم
// از جانب
// زاویه محور
  this.arch1.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 1, 0), Math.PI - rotation);
  this.arch2.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 1, 0), -rotation);

  world.addBody(this.arch1.body);
  world.addBody(this.arch2.body);
  if (debug) {
    addCannonVisual(this.arch1.body);
    addCannonVisual(this.arch2.body);
  }
};

/** این بخش دیوار است که موازی با محور x است */
var LongWall = function (x, y, z, width) {
  var height = 2;
  var thickness = 2.5;

  this.body = new CANNON.Body({
    mass: 0, // جرم == 0 بدن را ثابت می کند
    material: Table.wallContactMaterial
  });

  //تنظیم مختصات x برای تغییر زاویه شکل مثلث
  var vertices1 = [
    0,     height, -2 * thickness, // vertex 0
    0,     height,  0,             // vertex 1
    -2.8,  height, -2 * thickness, // vertex 2
    0,    -height, -2 * thickness, // vertex 3
    0,    -height,  0,             // vertex 4
    -2.8, -height, -2 * thickness  // vertex 5
  ];

  //گوشه میز
  var vertices2 = [
    0,  height, -2 * thickness, // vertex 0
    0,  height,  0,             // vertex 1
    8,  height, -2 * thickness, // vertex 2
    0, -height, -2 * thickness, // vertex 3
    0, -height,  0,             // vertex 4
    8, -height, -2 * thickness  // vertex 5
  ];

  var indices = [
    0, 1, 2,
    3, 4, 5,
    5, 0, 2,
    5, 3, 0,
    3, 4, 1,
    3, 1, 0,
    4, 5, 1,
    5, 2, 1
  ];
  //شکل سه بعدی
  var trimeshShape1 = new CANNON.Trimesh(vertices1, indices);
  var trimeshShape2 = new CANNON.Trimesh(vertices2, indices);

  this.body.position.set(x,y,z);
  this.body.addShape(trimeshShape1, new CANNON.Vec3(-width, 0, 0));
  this.body.addShape(trimeshShape2, new CANNON.Vec3( width, 0, 0));

  var boxshape = new CANNON.Box(new CANNON.Vec3(width, height, thickness));

  this.body.addShape(boxshape, new CANNON.Vec3(0 ,0, -thickness));
};

var WIDTH  = 1200,
    HEIGHT = 800;

// جهان سه بعدی
var renderer, scene, camera, game, controls, keyboard, lightsConfig, world, gui, eightballgame;
var debug = false; // اگر روشن باشد، فریم های برخوردش کشیده می شوند

var progressBar;

var stats = new Stats();
stats.setMode( 0 ); // 0: fps, 1: ms, 2: mb

var textureLoader = new THREE.TextureLoader();
THREE.DefaultLoadingManager.onProgress = function (item, loaded, total) {
  if (typeof progressBar !== 'undefined') {
    progressBar.style.width = (loaded / total * 100) + '%';
  }

  if (loaded == total && total > 7) {
    // پنهان كردن نوار پیشرفت
    var progBarDiv = document.getElementById('loading');
    progBarDiv.parentNode.removeChild(progBarDiv);

    gui.show(document.getElementById('mainMenu'));

    // عنصر DOM اجرا لوازم جانبی را ضمیمه کنید
    var canvasContainer = document.getElementById('canvas');
    canvasContainer.appendChild(renderer.domElement);
    draw();
  }
};

// برخی از ویژگی های دوربین را تنظیم کنید
var VIEW_ANGLE = 45,
    ASPECT     = WIDTH / HEIGHT,
    NEAR       = 1,
    FAR        = 1000;

// ما از ساعت برای اندازه گیری زمان، یک پسوند برای صفحه کلید استفاده می کنیم
var clock = new THREE.Clock();

function onLoad() {
  gui = new GameGui();

  progressBar = document.getElementById('prog-bar');

  // اجرایی WebGL، دوربین را ایجاد کنید
  // و یک صحنه
  camera = new THREE.PerspectiveCamera(VIEW_ANGLE, ASPECT, NEAR, FAR);
  camera.up = new THREE.Vector3(0,1,0);

  scene = new THREE.Scene();
  scene.add(camera);

  // ایجاد رندر
  renderer = new THREE.WebGLRenderer();
  renderer.setSize(WIDTH, HEIGHT);
  renderer.shadowMap.enabled = true;
  renderer.shadowMapSoft = true;

  // تنظیم world of your cannon.js برای فیزیک
  world = createPhysicsWorld();
  // جمعیت و فیزیک بدن:
  game = new Game();
  // تعریف کنید که چگونه اشیاء مختلف فیزیک را درک می کنند:
  setCollisionBehaviour();
  // نورپردازی را تنظیم کنید:
  addLights();

  // کنترل مویس
  controls = new THREE.OrbitControls(camera,renderer.domElement);

  controls.enableZoom = true;
  controls.enablePan = true;

  controls.minDistance = 35;
  controls.maxDistance = 165;
  // اجازه ندهید که دوربین به زیر زمین برود
  controls.maxPolarAngle = 0.49 * Math.PI;

  camera.position.set(-170, 70, 0);

  // پس زمینه رنگ سبز یشمی را عوض کنید.
  renderer.setClearColor(0x2727727, 1);
}

function createPhysicsWorld () {
  w = new CANNON.World();
  w.gravity.set(0, 30 * -9.82, 0); // m/s²

  w.solver.iterations = 10;
  // حل کننده نیروی
  //  برای استفاده از همه تکرارها
  w.solver.tolerance = 0;


  // اجازه وایسادن
  w.allowSleep = true;

  w.fixedTimeStep = 1.0 / 60.0; // seconds

  return w;
}

/** در اینجا تعامل زمانی که دو ماده لمس تعریف می شود تعریف می شود. به عنوان مثال. چقدر یک توپ
     هنگام برخورد با دیوار، انرژی را از دست می دهد.
    !TODO figure out whether the definition of Contactmaterials should be defined in
 جایی دیگر برای مثال شاید هر جسم توپ باید مواد تماس خود را با دیوارها تعریف کند.
*/
function setCollisionBehaviour() {
  world.defaultContactMaterial.friction = 0.1;
  world.defaultContactMaterial.restitution = 0.85;

  var ball_floor = new CANNON.ContactMaterial(
    Ball.contactMaterial,
    Table.floorContactMaterial,
    {friction: 0.7, restitution: 0.1}
  );

  var ball_wall = new CANNON.ContactMaterial(
    Ball.contactMaterial,
    Table.wallContactMaterial,
    {friction: 0.5, restitution: 0.9}
  );

  world.addContactMaterial(ball_floor);
  world.addContactMaterial(ball_wall);
}

function draw() {
  stats.begin();
  
  //کنترل
  controls.target.copy(game.balls[0].mesh.position);
  controls.update();

  // جهان فیزیک
  world.step(w.fixedTimeStep);

  // سه اشیاء
  var dt = clock.getDelta();
  game.tick(dt);

  stats.end();
  requestAnimationFrame(draw);
  renderer.render(scene, camera); //صحنه های ما را با دوربین ما ترسیم می کنیم
}

// نور محیط و دو نقطه نور بالا را بالای میز اضافه می کند
function addLights() {
  var light = new THREE.AmbientLight(0x0d0d0d); //نور محیط لطیف سفید
  scene.add(light);
  var tableLight1 = new TableLight( Table.LEN_X / 4, 150, 0);
  var tableLight2 = new TableLight(-Table.LEN_X / 4, 150, 0);
}

/** این بخش دیوار است که موازی با محور z است*/
var ShortWall = function (x, y, z, width) {
  var height = 2;
  var thickness = 4;
  this.body = new CANNON.Body({
    mass: 0, // جرم == 0 بدن را ثابت می کند
    material: Table.wallContactMaterial
  });

  // چگونه یک مش با یک مثلث واحد ایجاد کنیم
  var vertices1 = [
        0,  height, -2 * thickness, // vertex 0
        0,  height,  0,             // vertex 1
    -12.5,  height, -2*thickness,   // vertex 2
        0, -height, -2*thickness,   // vertex 3
        0, -height,  0,             // vertex 4
    -12.5, -height, -2*thickness    // vertex 5
  ];

  // گوشه میز
  var vertices2 = [
       0,  height, -2 * thickness,  // vertex 0
       0,  height,  0,              // vertex 1
    12.5,  height, -2 * thickness,  // vertex 2
       0, -height, -2 * thickness,  // vertex 3
       0, -height,  0,              // vertex 4
    12.5, -height, -2 * thickness   // vertex 5
  ];

  var indices = [
    0, 1, 2,
    3, 4, 5,
    5, 0, 2,
    5, 3, 0,
    3, 4, 1,
    3, 1, 0,
    4, 5, 1,
    5, 2, 1
  ];
  var trimeshShape1 = new CANNON.Trimesh(vertices1, indices);
  var trimeshShape2 = new CANNON.Trimesh(vertices2, indices);

  this.body.position.set(x,y,z);
  this.body.addShape(trimeshShape1, new CANNON.Vec3(-width, 0, 0));
  this.body.addShape(trimeshShape2, new CANNON.Vec3( width, 0, 0));

  var boxshape = new CANNON.Box(new CANNON.Vec3(width, height, thickness));

  this.body.addShape(boxshape, new CANNON.Vec3(0 ,0, -thickness));

  this.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 1, 0), -Math.PI / 2);
};

var Table = function () {
  var mesh_x = -Table.LEN_X / 2;
  var mesh_y = 0;
  var mesh_z = Table.LEN_Z / 2;

  var loader = new THREE.JSONLoader();
  loader.load('json/table/base.json', function (geometry) {
    var mesh = new THREE.Mesh(geometry, new THREE.MeshPhongMaterial({
      color: new THREE.Color(0x000000),
      specular: 0x404040,
      shininess: 25,
      shading: THREE.SmoothShading
    }));

    mesh.position.x = mesh_x;
    mesh.position.y = mesh_y;
    mesh.position.z = mesh_z;
    mesh.scale.set(100, 100, 100);
    mesh.receiveShadow = true;
    mesh.castShadow = true;
    scene.add(mesh);
  });

  loader.load('json/table/felt.json', function (geometry) {
    var mesh = new THREE.Mesh(geometry, new THREE.MeshPhongMaterial({
      color: new THREE.Color(TABLE_COLORS.cloth),
      specular: 0x505050,
      shininess: 10,
      shading: THREE.SmoothShading
    }));

    mesh.position.x = mesh_x;
    mesh.position.y = mesh_y;
    mesh.position.z = mesh_z;
    mesh.scale.set(100, 100, 100);
    mesh.receiveShadow = true;
    mesh.castShadow = true;
    scene.add(mesh);
  });

  loader.load('json/table/edges.json', function (geometry) {
    var mesh = new THREE.Mesh(geometry, new THREE.MeshPhongMaterial({
      color: new THREE.Color(0x8a5230),
      specular: 0x505050,
      shininess: 100,
      shading: THREE.SmoothShading
    }));

    mesh.position.x = mesh_x;
    mesh.position.y = mesh_y;
    mesh.position.z = mesh_z;
    mesh.scale.set(100, 100, 100);
    mesh.receiveShadow = true;
    mesh.castShadow = true;
    scene.add(mesh);
  });

  loader.load('json/table/pockets.json', function (geometry) {
    var mesh = new THREE.Mesh(geometry, new THREE.MeshPhongMaterial({
      color: new THREE.Color(0x8a5230),
      specular: 0x4D4D4D,
      shininess: 25,
      shading: THREE.SmoothShading
    }));

    mesh.position.x = mesh_x;
    mesh.position.y = mesh_y;
    mesh.position.z = mesh_z;
    mesh.scale.set(100, 100, 100);
    mesh.receiveShadow = true;
    mesh.castShadow = true;
    scene.add(mesh);
  });

  loader.load('json/table/pocket_bottoms.json', function (geometry) {
    var mesh = new THREE.Mesh(geometry, new THREE.MeshPhongMaterial({
      color: new THREE.Color(0x000),
      specular: 0x000,
      shininess: 0,
      shading: THREE.SmoothShading
    }));

    mesh.position.x = mesh_x;
    mesh.position.y = mesh_y;
    mesh.position.z = mesh_z;
    mesh.scale.set(100, 100, 100);
    mesh.receiveShadow = true;
    mesh.castShadow = true;
    scene.add(mesh);
  });

  this.rigidBody = this.createFloor();

  //corners of -x table side
  this.hole1 = new Hole( Table.LEN_X / 2 + 1.5, 0, -Table.LEN_Z / 2 - 1.5,  Math.PI / 4);
  this.hole2 = new Hole(-Table.LEN_X / 2 - 1.5, 0, -Table.LEN_Z / 2 - 1.5, -Math.PI / 4);
  //middle holes
  this.hole3 = new Hole(0, 0, -Table.LEN_Z / 2 - 4.8, 0);
  this.hole4 = new Hole(0, 0,  Table.LEN_Z / 2 + 4.8, Math.PI);
  //corners of +x table side
  this.hole5 = new Hole( Table.LEN_X / 2 + 1.5, 0, Table.LEN_Z / 2 + 1.5,  3 * Math.PI / 4);
  this.hole6 = new Hole(-Table.LEN_X / 2 - 1.5, 0, Table.LEN_Z / 2 + 1.5, -3 * Math.PI / 4);

  this.walls = this.createWallBodies();
};

var TABLE_COLORS = {
  cloth: 0x4d9900
};

Table.LEN_Z = 137.16;
Table.LEN_X = 274.32;
Table.WALL_HEIGHT = 6;
Table.floorContactMaterial = new CANNON.Material('floorMaterial');
Table.wallContactMaterial = new CANNON.Material('wallMaterial');

/** Creates cannon js walls
This method is 3am spaghetti, you've been warned..*/
Table.prototype.createWallBodies = function () {
  //walls of -z
  var wall1 = new LongWall( Table.LEN_X / 4 - 0.8, 2, -Table.LEN_Z / 2, 61);
  var wall2 = new LongWall(-Table.LEN_X / 4 + 0.8, 2, -Table.LEN_Z / 2, 61);
  wall2.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 0, 1), Math.PI);

  //walls of -z
  var wall3 = new LongWall( Table.LEN_X / 4 - 0.8, 2, Table.LEN_Z / 2, 61);
  var wall4 = new LongWall(-Table.LEN_X / 4 + 0.8, 2, Table.LEN_Z / 2, 61);
  wall3.body.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0),  Math.PI);
  wall4.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 1, 0), -Math.PI);

  //wall of +x
  var wall5 = new ShortWall(Table.LEN_X / 2, 2, 0, 60.5);

  //wall of -x
  var wall6 = new ShortWall(-Table.LEN_X / 2, 2, 0, 60.5);
  wall6.body.quaternion.setFromAxisAngle(new CANNON.Vec3(0, 1, 0), -1.5 * Math.PI);

  var walls = [wall1, wall2, wall3, wall4, wall5, wall6];
  for (var i in walls) {
    world.addBody(walls[i].body);
    if (debug) {
      addCannonVisual(walls[i].body);
    }
  }

  return walls;
};

Table.prototype.createFloor = function () {
  var narrowStripWidth = 2;
  var narrowStripLength = Table.LEN_Z / 2 - 5;
  var floorThickness = 1;
  var mainAreaX = Table.LEN_X / 2 - 2 * narrowStripWidth;

  var floorBox = new CANNON.Box(new CANNON.Vec3(mainAreaX, floorThickness, Table.LEN_Z / 2));
  var floorBoxSmall = new CANNON.Box(new CANNON.Vec3(narrowStripWidth, floorThickness, narrowStripLength));

  this.body = new CANNON.Body({
    mass: 0, // mass == 0 makes the body static
    material: Table.floorContactMaterial
  });
  this.body.addShape(floorBox,      new CANNON.Vec3(0, -floorThickness, 0));
  this.body.addShape(floorBoxSmall, new CANNON.Vec3(-mainAreaX - narrowStripWidth, -floorThickness, 0));
  this.body.addShape(floorBoxSmall, new CANNON.Vec3( mainAreaX + narrowStripWidth, -floorThickness, 0));

  if (debug) {
    addCannonVisual(this.body, 0xff0000);
  }
  world.add(this.body);
};

var TableLight = function (x,y,z) {
  this.spotlight = new THREE.SpotLight(0xffffe7, 1);

  this.spotlight.position.set(x, y, z);
  this.spotlight.target.position.set(x, 0, z); //the light points directly towards the xz plane
  this.spotlight.target.updateMatrixWorld();

  this.spotlight.castShadow = true;
  this.spotlight.shadowCameraFov = 120;
  this.spotlight.shadowCameraNear = 100;
  this.spotlight.shadowCameraFar = 180;
  this.spotlight.shadowMapWidth = 2048;
  this.spotlight.shadowMapHeight = 2048;

  scene.add(this.spotlight);

  if (debug) {
    this.shadowCam = new THREE.CameraHelper(this.spotlight.shadow.camera);
    scene.add(this.shadowCam);
  }
};

var WhiteBall = function (x, y, z) {
  this.color = 0xffffff;
  this.defaultPosition = new CANNON.Vec3(-Table.LEN_X / 4, Ball.RADIUS, 0);
  // Call the parent constructor, making sure (using Function#call)
  // that "this" is set correctly during the call
  Ball.call(
    this,
    this.defaultPosition.x,
    this.defaultPosition.y,
    this.defaultPosition.z,
    'whiteball',
    this.color
  );

  this.forward = new THREE.Vector3(1, 0, 0);
  this.forwardLine = this.createForwardLine();
  scene.add(this.forwardLine);

  this.dot = this.createIntersectionDot();
  scene.add(this.dot);
};

WhiteBall.prototype = Object.create(Ball.prototype);
WhiteBall.prototype.constructor = WhiteBall;

/** Applies a force to this ball to make it move.
    The strength of the force is given by the argument
    The force is the balls "forward" vector, applied at the
    edge of the ball in the opposite direction of the "forward"
*/
WhiteBall.prototype.hitForward = function (strength) {
  this.rigidBody.wakeUp();
  var ballPoint = new CANNON.Vec3();
  ballPoint.copy(this.rigidBody.position);

  var vec = new CANNON.Vec3();
  vec.copy(this.forward);

  vec.normalize();
  vec.scale(Ball.RADIUS, vec);
  ballPoint.vsub(vec, ballPoint);

  var force = new CANNON.Vec3();
  force.copy(this.forward.normalize());
  force.scale(strength, force);
  this.rigidBody.applyImpulse(force, ballPoint);
};

/** Resets the position to this.defaultPosition */
WhiteBall.prototype.onEnterHole = function () {
  this.rigidBody.velocity = new CANNON.Vec3(0);
  this.rigidBody.angularVelocity = new CANNON.Vec3(0);
  this.rigidBody.position.copy(this.defaultPosition);
  eightballgame.whiteBallEnteredHole();
};

WhiteBall.prototype.tick = function (dt) {
  //Superclass tick behaviour:
  Ball.prototype.tick.apply(this, arguments);

  //update intersection dot if were not moving
  if (this.rigidBody.sleepState == CANNON.Body.SLEEPING) {
    if (!this.forwardLine.visible) {
      this.forwardLine.visible = true;
    }
    if (!this.dot.visible) {
      this.dot.visible = true;
    }
    this.updateGuideLine();
    this.updateIntersectionDot();
  } else {
    if (this.forwardLine.visible) {
      this.forwardLine.visible = false;
    }
    if (this.dot.visible) {
      this.dot.visible = false;
    }
  }
};

WhiteBall.prototype.createIntersectionDot = function () {
  var geometry = new THREE.SphereGeometry(1, 4, 4);
  var material = new THREE.MeshBasicMaterial({opacity: 0.5, transparent: true, color: 0xfff003});
  var sphere = new THREE.Mesh(geometry, material);

  return sphere;
};
WhiteBall.prototype.updateIntersectionDot = function () {
  this.dot.position.copy(this.intersectionPoint);
};

WhiteBall.prototype.updateGuideLine = function () {
  var angle = controls.getAzimuthalAngle() + Math.PI / 2;
  this.forward.set(Math.cos(angle), 0, -Math.sin(angle));

  this.forwardLine.position.copy(this.mesh.position);

  this.forwardLine.rotation.y = angle;
  this.forward.normalize();

  // Go through each ball
  var distances = [];
  for (var i = 1; i < game.balls.length; i++) {
    // find the distance to that ball
    distances.push({
      index: i,
      dist: Math.abs(this.mesh.position.distanceTo(game.balls[i].mesh.position))
    });
  }

  //sort the according to distance
  distances.sort(function (a, b) { return a.dist - b.dist; });

  //iterate again, to find the closest intersecting ball
  var intersectingBallIndex = -1;
  for (var j = 0; j < distances.length; j++) {
    var ballIndex = distances[j].index;
    var curBall = game.balls[ballIndex];
    if (this.forwardLine.ray.isIntersectionSphere(curBall.sphere)) {
      intersectingBallIndex = ballIndex;
      break;
    }
  }
  //This could possibly be optimized with some more clever usage of THREE js-s offered functions (look into Ray, etc)

  if (intersectingBallIndex == -1) {
    // We're intersecting with the edge of the table
    this.intersectionPoint = this.forwardLine.ray.intersectBox(this.forwardLine.box);
  } else {
    // Otherwise we are aiming at some ball
    this.intersectionPoint = this.forwardLine.ray.intersectSphere(game.balls[intersectingBallIndex].sphere);
  }

  var distance = Math.sqrt(this.mesh.position.distanceToSquared(this.intersectionPoint));

  this.forwardLine.geometry.vertices[1].x = distance;
  this.forwardLine.geometry.verticesNeedUpdate = true;
};

WhiteBall.prototype.createForwardLine = function () {
  var lineGeometry = new THREE.Geometry();
  var vertArray = lineGeometry.vertices;

  vertArray.push(new THREE.Vector3(0, 0, 0));
  vertArray.push(new THREE.Vector3(85, 0, 0));
  lineGeometry.computeLineDistances();
  var lineMaterial = new THREE.LineDashedMaterial({ color: 0xdddddd, dashSize: 4, gapSize: 2 });
  var line = new THREE.Line(lineGeometry, lineMaterial);
  line.position.copy(new THREE.Vector3(100, 100, 100)); //hide it somewhere initially
  line.box = new THREE.Box3(
    new THREE.Vector3(-Table.LEN_X / 2, 0,               -Table.LEN_Z / 2),
    new THREE.Vector3( Table.LEN_X / 2, 2 * Ball.RADIUS,  Table.LEN_Z / 2)
  );

  line.ray = new THREE.Ray(this.mesh.position, this.forward);

  return line;
};
