<template>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import * as THREE from "three"

//TEXTURE
//Ci  sono vari tipi di texture:
// Color Texture (Or Albedo Texture) => é Semplicemente un colore che verrà applicato alla geometria
// Alpha Texture => è un immagine in scale di grigi, la parte bianca è visibile, la parte nera non è vibile (grigio è quasi vibile, a seconda di quanto scuro)
// Height Texture (or Displacement) => è sempre un immagine in scala di grigi, e puo muovere i vertici (per creare rilievi), se è bianco il vertice sara spostato in alto, nero in basso, mentre se è del grigio perfetto(a meta tra nero e bianco) il vertice non si muovera.
// Normal Texture => aggiunge dettagli (maggiormente per quano riguarda la luce), non ha bisogno di sudivisione perche non muove i vertici, ed ha performance migliori rispetto aggiungere una height texture con molte suddivisioni
// Ambient Occlusion Texture => Sempre in scala di grigi, aggiunge "false" ombre nelle fessure (dove ci sono angoli, aggiunge ombre), non è fisicamente accurato, aiuta a creare contrasti ed aggiungere dettagli
// Metalness Texture => Sempre in scala di grigi, quando è bianca è metallic, quando è nera è non metallica. viene usata maggiormente per creare riflessi. quando è metallica possiamo vedere il riflesso in questo oggetto
// Roughness Texture => sempre in scala di grigi, funziona in combinazione con metallness (non sempre ma spesso usate insieme), usata maggiormente per la dissipazione di luce. bianco è "duro", nero è "leggero". in pratica dissipa la luce se è su bianco, rimbalza la luce se è nero


//Ma come carichiamo le immagini? ci sono piu modi di farlo:

/*1) NATIVE JAVASCRIPT
const image = new Image() //crea un immagine vuota

//ma comunque non possiamo utilizzare direttamente l'immagine, abbiamo bisogno di trasformarla prima in una texture
//con la classe Texture di threejs,in questo modo l' immagine verra convertito in un formato piu "GPU Friendly" e che potra essere utilizzato per il WebGL Render
const texture = new THREE.Texture(image)

image.onload = ()=>{ //fa qualcosa quando l'immagine è stata caricata
  
  //aggiorniamo la texture quando viene caricata una nuova immagine
  texture.needsUpdate = true 
 
  console.log("image loaded: ", texture)
}

image.src = "./texture/cube.png" //aggiungiamo il percorso dell'immagine
*/


/*2) THREEJS TEXTURELOADER
const textureLoader = new THREE.TextureLoader()
//const texture = textureLoader.load("/texture/cube.png")

//possiamo passare 3 diverse funzioni dopo il path: onLoad(immagine caricata con successo), onProgress(progresso durante il caricamento), e onError ( se escono errori)
const texture = textureLoader.load(
  "/texture/cube.png",
  
  //onLoad
  ()=>{
    console.log("image loaded")
  },
  
  //onProgress
  ()=>{
    console.log("in progress...")
  },

  //onError
  ()=>{
    console.log("Error!11!! KAffE'è?!")
  }
)
*/

// TEXTURELOADER + LOADING MANAGER

//Il loading manager ci permette di essere al corrente dello stato di progresso del "loading" globale,
//o essere informati quando tuto quanto è stato caricato
const loadingManager = new THREE.LoadingManager()

//per essere informato sugli stati di caricamento questa volta impostiamo i metodi propri del loading manager
loadingManager.onStart = () =>{
  console.log("loading started")
}

loadingManager.onProgress = () =>{
  console.log("loading...")
}

loadingManager.onLoad = () =>{
  console.log("loading complete")
}

loadingManager.onError = () =>{
  console.log("loading error")
}


//Per fare cio basta inserire il loading manager come argomento del texure loader
const textureLoader = new THREE.TextureLoader(loadingManager)

//possiamo passare 3 diverse funzioni dopo il path: onLoad(immagine caricata con successo), onProgress(progresso durante il caricamento), e onError ( se escono errori)
const colorTexture = textureLoader.load("/texture/cube.png")

//Carichiamo altre texture
const cubeTexture = textureLoader.load("/texture/cube1.jpg")

//TRASFORMATIONS
//possiamo cambiare anche modificare le proprietà delle texture ad esempio:

//ripetiamo la texture due volte su asse x e 3 volte su asse y
//colorTexture.repeat.x = 2
//colorTexture.repeat.y = 3
//colorTexture.wrapS = THREE.RepeatWrapping
//colorTexture.wrapT = THREE.MirroredRepeatWrapping

//cambiamo l'offset della texture (la posizione)
//colorTexture.offset.x = 0.5
//colorTexture.offset.y = 0.5

//ruotiamo la texture
colorTexture.rotation = Math.PI / 4

//cambiamo il punto in cui "parte" la rotazione della texture, impostando il centro dell'oggetto su cui applichiamo la texture come punto da cui parte la rotazione
colorTexture.center.x = 0.5
colorTexture.center.y = 0.5

//tips:
//cerca di mantenere le immagini delle texture il piu piccole possibile a livello di dimensioni, senza perdere qualità. in questo modo sarà piu veloce da caricare ed offrirai un esperienza utente migliore
//Differenza tra png e jpg
//JPG:
//- piu leggere ma perdono informazioni
//- non supportano la trasparenza, se la si vuole implementare bisogna utilizzare 2 texure , una per il color e l'altra per l'alpha
//- le normal texture in jpg hanno un effetto "blur" e altri strani effetti dovuti alla perdita d'informazione

//PNG: 
//- piu dettagliate ma piu pesanti
//- supportano la trasparenza, quind se vuoi usare solo una texure per color e alpha scegli la png
//- supporta le normal texture in quanto non ha (grosse) perdite d'informazioni


//Siti web dove trovare texture gratis (e a pagamento): 
//- poliigon.com
//- 3dtextures.me
//- arroway-textures.ch

var controls

let canvasRef = ref();

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{

  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

//SCENE
let scene = new THREE.Scene()

//RENDER
let renderer

//AGGIUNGIAMO LA TEXTURE AL BOX

// BOX
const boxGeometry = new THREE.BoxGeometry(1,1,1)
const boxMaterial = new THREE.MeshBasicMaterial({map: colorTexture}) 
const box = new THREE.Mesh(boxGeometry,boxMaterial)
box.position.set(0 , 0, 0)
scene.add(box)





// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);

camera.position.set(0,0,3)
camera.lookAt(box.position)
scene.add(camera);

const clock = new THREE.Clock()

const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value
  });
 
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) 
  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true;
});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>