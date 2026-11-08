# enjambre-sonoro-unidad-3
// PROYECTO UNIDAD 3 - ENJAMBRE SONORO
// Nombres: Juan Pablo Barriga, Pablo Cortez, Sebastian Bollo, Benjamin Rozas
//vFinal con dashboard agregado
// El microfono mide amplitud; no se guarda ni se envia audio.


// MICROFONO Y SIMULACION

let fuente = "micro";
let mic;
let microSolicitado = false;
let falla = false;

function setupMic() {
  mic = new p5.AudioIn();
}

// El navegador pide un clic para activar el audio.
function mousePressed() {
  userStartAudio();
  if (!microSolicitado) {
    mic.start();
    microSolicitado = true;
  }
}

function leerCrudo() {
  // La tecla F simula una caida de la fuente.
  if (falla) {
    return null;
  }

  if (fuente === "micro") {
    let nivel = mic.getLevel();

    
    if (nivel === 0) {
      return null;
    }
    return nivel;
  } else {
    // Misma secuencia si se repite con noiseSeed(42).
    let tiempo = frameCount * 0.02;
    let valor = noise(tiempo);

    if (valor < 0.35) {
      return 0.002; // sonido bajo
    } else if (valor < 0.65) {
      return 0.035; // conversacion
    } else {
      return 0.120; // ruido fuerte
    }
  }
}

function keyPressed() {
  if (key === "m" || key === "M") {
    if (fuente === "micro") {
      fuente = "simulacion";
    } else {
      fuente = "micro";
    }
    console.log("Fuente: " + fuente);
  }

  if (key === "f" || key === "F") {
    falla = !falla;
    console.log("Falla simulada: " + falla);
  }
}


//  CALIBRACION Y ALERTAS

let minimoMedido = 0.002;
let maximoMedido = 0.120;

let suavidad = 0.15;
let senalSuavizada = 0;
let umbralActivar = 0.70;
let umbralDesactivar = 0.50;
let alertaActiva = false;
let fallaPersona2 = false;

function procesarSeñal() {
  let crudo = leerCrudo();

  if (crudo === null) {
    fallaPersona2 = true;
    // No se inventa un dato: la normalizada se marca con null.
    // Se conserva el ultimo valor suavizado, pero no se actualiza.
    return {
      normalizada: null,
      suavizada: senalSuavizada,
      alerta: alertaActiva,
      falla: true
    };
  }

  fallaPersona2 = false;

  let normalizada = map(crudo, minimoMedido, maximoMedido, 0, 1);
  normalizada = constrain(normalizada, 0, 1);

  senalSuavizada = lerp(senalSuavizada, normalizada, suavidad);

  if (!alertaActiva && senalSuavizada >= umbralActivar) {
    alertaActiva = true;
  } else if (alertaActiva && senalSuavizada <= umbralDesactivar) {
    alertaActiva = false;
  }

  return {
    normalizada: normalizada,
    suavizada: senalSuavizada,
    alerta: alertaActiva,
    falla: false
  };
}


// ENJAMBRE DE PARTICULAS

let particulas = [];
let cantidadParticulas = 60;
let nivelEnjambre = 0;
let fallaEnjambre = false;

function crearEnjambre() {
  particulas = [];
  for (let i = 0; i < cantidadParticulas; i++) {
    particulas.push({
      x: random(width),
      y: random(height),
      vx: random(-1, 1),
      vy: random(-1, 1),
      tamano: random(4, 8)
    });
  }
}

function actualizarEnjambre(datos) {
  fallaEnjambre = datos.falla;
  if (fallaEnjambre) {
    return; // En una falla, el enjambre queda quieto.
  }
  nivelEnjambre = datos.suavizada;
  moverParticulas();
}

function moverParticulas() {
  let velocidad = map(nivelEnjambre, 0, 1, 0.5, 4);
  let agitacion = map(nivelEnjambre, 0, 1, 0.02, 0.25);
  let atraccion = map(nivelEnjambre, 0, 1, 0.012, 0.001);

  for (let p of particulas) {
    p.vx += (width / 2 - p.x) * atraccion * 0.01;
    p.vy += (height / 2 - p.y) * atraccion * 0.01;

    p.vx += random(-agitacion, agitacion);
    p.vy += random(-agitacion, agitacion);

    let rapidez = sqrt(p.vx * p.vx + p.vy * p.vy);
    if (rapidez > velocidad) {
      p.vx = p.vx / rapidez * velocidad;
      p.vy = p.vy / rapidez * velocidad;
    }

    p.x += p.vx;
    p.y += p.vy;
    controlarBordes(p);
  }
}

function controlarBordes(p) {
  let radio = p.tamano / 2;

  if (p.x < radio) {
    p.x = radio;
    p.vx = abs(p.vx);
  }
  if (p.x > width - radio) {
    p.x = width - radio;
    p.vx = -abs(p.vx);
  }
  if (p.y < radio) {
    p.y = radio;
    p.vy = abs(p.vy);
  }
  if (p.y > height - radio) {
    p.y = height - radio;
    p.vy = -abs(p.vy);
  }
}

function dibujarEnjambre() {
  noStroke();
  if (fallaEnjambre) {
    fill(140); // Gris: sin datos
  } else if (nivelEnjambre < 0.30) {
    fill(80, 205, 240); // Celeste: bajo
  } else if (nivelEnjambre < 0.65) {
    fill(245, 190, 65); // Amarillo: moderado
  } else {
    fill(240, 90, 95); // Rojo: alto
  }

  for (let p of particulas) {
    circle(p.x, p.y, p.tamano);
  }
}


// INTEGRACION setup y draw

function setup() {
  createCanvas(700, 450);
  noiseSeed(42);
  randomSeed(42);
  setupMic();
  crearEnjambre();
}

function draw() {

  background(20, 28, 40);

  let datos = procesarSeñal();

  actualizarEnjambre(datos);

  dibujarEnjambre();

  dibujarDashboard(datos);
}



//  DASHBOARD


function dibujarDashboard(datos) {

  // Posicion y tamaño del panel
  let x = width - 255;
  let y = 15;

  // Fondo del dashboard
  noStroke();
  fill(15, 23, 36);
  rect(x, y, 240, 420);

  // Titulo
  fill(255);
  textSize(18);
  text("ENJAMBRE SONORO", x + 15, y + 30);

  textSize(11);
  fill(180, 190, 205);
  text("Monitor de ruido ambiental", x + 15, y + 50);

  // Mostrar la fuente
  let nombreFuente = "Microfono";

  if (fuente === "simulacion") {
    nombreFuente = "Simulacion";
  }

  fill(255);
  textSize(13);
  text("Fuente: " + nombreFuente, x + 15, y + 80);

  // Comprobar si existe una falla
  if (datos.falla) {

    fill(240, 90, 95);
    text("ESTADO: SIN DATOS", x + 15, y + 110);

    fill(255);
    text("Intensidad: --", x + 15, y + 145);

    fill(55, 65, 80);
    rect(x + 15, y + 160, 210, 15);

    fill(180);
    text("Alerta: sin datos", x + 15, y + 205);

  } else {

    // Determinar estado segun la señal
    let estado = "";
    let r = 0;
    let g = 0;
    let b = 0;

    if (datos.suavizada < 0.30) {

      estado = "TRANQUILO";
      r = 80;
      g = 205;
      b = 240;

    } else if (datos.suavizada < 0.65) {

      estado = "MODERADO";
      r = 245;
      g = 190;
      b = 65;

    } else {

      estado = "ELEVADO";
      r = 240;
      g = 90;
      b = 95;
    }

    // Mostrar estado
    fill(r, g, b);
    text("Estado: " + estado, x + 15, y + 110);

    // Mostrar intensidad relativa
    let porcentaje = round(datos.suavizada * 100);

    fill(255);
    text("Intensidad: " + porcentaje + "%",
         x + 15, y + 145);

    // Fondo de la barra
    fill(55, 65, 80);
    rect(x + 15, y + 160, 210, 15);

    // Barra que crece con el sonido
    fill(r, g, b);
    rect(
      x + 15,
      y + 160,
      210 * datos.suavizada,
      15
    );

    // Mostrar alerta
    if (datos.alerta) {

      fill(240, 90, 95);
      text("ALERTA: ACTIVADA", x + 15, y + 205);

    } else {

      fill(100, 220, 150);
      text("Alerta: desactivada", x + 15, y + 205);
    }
  }

  
  // LEYENDA DE COLORES

  fill(255);
  textSize(14);
  text("Referencia visual", x + 15, y + 250);

  textSize(12);

  // Sonido bajo
  fill(80, 205, 240);
  circle(x + 25, y + 280, 10);

  fill(255);
  text("Sonido bajo", x + 40, y + 284);

  // Sonido moderado
  fill(245, 190, 65);
  circle(x + 25, y + 305, 10);

  fill(255);
  text("Sonido moderado", x + 40, y + 309);

  // Sonido elevado
  fill(240, 90, 95);
  circle(x + 25, y + 330, 10);

  fill(255);
  text("Sonido elevado", x + 40, y + 334);

  // Instrucciones
  fill(170, 180, 195);
  textSize(11);

  text("M: Cambiar fuente", x + 15, y + 380);
  text("F: Simular falla", x + 15, y + 400);
}
