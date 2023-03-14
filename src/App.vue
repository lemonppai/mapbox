<template>
  <div id="map"></div>
</template>

<script>
  import mapboxgl from 'mapbox-gl/dist/mapbox-gl-dev';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
  import 'mapbox-gl/dist/mapbox-gl.css';
  import * as THREE from 'three';

  // console.log(THREE)

  export default {
    mounted() {
      this.initMap();
      this.renderThreeModel();
    },

    methods: {
      initMap() {
        const MAPTYPE = {
          NAVIMG: 'nav_img',
          NAVLBL: 'nav_lbl',
          NAVVEC: 'nav_vec'
        }

        function getNavTileUrls(lyr) {
          let res = [];
          const dict = {
            'nav_img': 'https://webst0${subdomain}.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}',
            'nav_lbl': 'https://webst0${subdomain}.is.autonavi.com/appmaptile?style=8&x={x}&y={y}&z={z}',
            'nav_vec': 'http://webrd0${subdomain}.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scale=1&style=8'
          }
          for (let i = 1; i < 5; i++) {
            let url = dict[lyr];
            url = url.split('${subdomain}').join(i);
            res.push(url)
          }
          return res;
        }

        const mapStyle = {
          "version": 8,
          "name": "Dark",
          "sources": {
            "osm": {
              "type": "raster",
              "tiles": [
                'https://tile.openstreetmap.org/{z}/{x}/{y}.png'
              ],
              "tileSize": 256
            },
            "gaode-img": {
              "type": "raster",
              "tiles": getNavTileUrls(MAPTYPE.NAVIMG),
              "tileSize": 256
            },
            "gaode-lbl": {
              "type": "raster",
              "tiles": getNavTileUrls(MAPTYPE.NAVLBL),
              "tileSize": 256
            },
            "gaode-vec": {
              "type": "raster",
              "tiles": getNavTileUrls(MAPTYPE.NAVVEC),
              "tileSize": 256
            }
          },
          "layers": [{
            "id": "img",
            "type": "raster",
            "source": "gaode-vec",
            "minzoom": 1,
            "maxzoom": 18
          }]
        };

        this.map = window.map = new mapboxgl.Map({
          container: 'map',
          style: mapStyle,
          hash: false,
          center: [104.61023753726323, 35.63101027697721],
          zoom: 14,
          // maxZoom: 17.2
          pitch: 45,
          bearing: -17.6
        });

        this.map.addControl(new mapboxgl.NavigationControl());
        this.map.addControl(new mapboxgl.FullscreenControl());
      },

      renderThreeModel() {
        // parameters to ensure the model is georeferenced correctly on the map
        const modelOrigin = [104.61023753726323, 35.63101027697721];
        const modelAltitude = 0;
        const modelRotate = [Math.PI / 2, 0, 0];

        const modelAsMercatorCoordinate = mapboxgl.MercatorCoordinate.fromLngLat(
          modelOrigin,
          modelAltitude
        );

        // transformation parameters to position, rotate and scale the 3D model onto the map
        const modelTransform = {
          translateX: modelAsMercatorCoordinate.x,
          translateY: modelAsMercatorCoordinate.y,
          translateZ: modelAsMercatorCoordinate.z,
          rotateX: modelRotate[0],
          rotateY: modelRotate[1],
          rotateZ: modelRotate[2],
          /* Since the 3D model is in real world meters, a scale transform needs to be
           * applied since the CustomLayerInterface expects units in MercatorCoordinates.
           */
          scale: modelAsMercatorCoordinate.meterInMercatorCoordinateUnits() * 3
        };

        // configuration of the custom layer for a 3D model per the CustomLayerInterface
        const customLayer = {
          id: '3d-model',
          type: 'custom',
          renderingMode: '3d',
          onAdd: function (map, gl) {
            this.camera = new THREE.Camera();
            this.scene = new THREE.Scene();

            // create two three.js lights to illuminate the model
            const directionalLight = new THREE.DirectionalLight(0xffffff);
            directionalLight.position.set(0, -70, 100).normalize();
            this.scene.add(directionalLight);

            const directionalLight2 = new THREE.DirectionalLight(0xffffff);
            directionalLight2.position.set(0, 70, 100).normalize();
            this.scene.add(directionalLight2);

            // use the three.js GLTF loader to add the 3D model to the three.js scene
            const loader = new GLTFLoader();
            loader.load(
              'https://docs.mapbox.com/mapbox-gl-js/assets/34M_17/34M_17.gltf',
              (gltf) => {
                this.scene.add(gltf.scene);
              }
            );
            this.map = map;

            // use the Mapbox GL JS map canvas for three.js
            this.renderer = new THREE.WebGLRenderer({
              canvas: map.getCanvas(),
              context: gl,
              antialias: true
            });

            this.renderer.autoClear = false;
          },
          render: function (gl, matrix) {
            // console.log(matrix)
            const rotationX = new THREE.Matrix4().makeRotationAxis(
              new THREE.Vector3(1, 0, 0),
              modelTransform.rotateX
            );
            const rotationY = new THREE.Matrix4().makeRotationAxis(
              new THREE.Vector3(0, 1, 0),
              modelTransform.rotateY
            );
            const rotationZ = new THREE.Matrix4().makeRotationAxis(
              new THREE.Vector3(0, 0, 1),
              modelTransform.rotateZ
            );

            const m = new THREE.Matrix4().fromArray(matrix);
            const l = new THREE.Matrix4()
              .makeTranslation(
                modelTransform.translateX,
                modelTransform.translateY,
                modelTransform.translateZ
              )
              .scale(
                new THREE.Vector3(
                  modelTransform.scale,
                  -modelTransform.scale,
                  modelTransform.scale
                )
              )
              .multiply(rotationX)
              .multiply(rotationY)
              .multiply(rotationZ);

            this.camera.projectionMatrix = m.multiply(l);
            this.renderer.resetState();
            this.renderer.render(this.scene, this.camera);
            this.map.triggerRepaint();
          }
        };

        this.map.on('style.load', () => {
          this.map.addLayer(customLayer);

          // create the popup
          /* const popup = new mapboxgl.Popup({ offset: 25 })
            .setHTML('经纬度：<br>104.61023753726323, 35.63101027697721');

          const marker1 = new mapboxgl.Marker()
            .setLngLat([104.61023753726323, 35.63101027697721])
            .setPopup(popup)
            .addTo(this.map); */
        });
      }
    }
  }

</script>

<style>
  body {
    margin: 0;
    padding: 0;
  }

  #map {
    height: 100vh;
  }

</style>
