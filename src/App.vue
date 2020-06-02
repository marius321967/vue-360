<template>
  <div id="app">
    <div class="container">
      <div class="row">
        
        <!-- Example 1 - Car -->
        <div class="col-12 mb-4 card p-0">
          <vue-three-sixty 
            :amount=180
            :layers="layers"
            spinReverse
            autoplay
            buttonClass="dark"
          >
            <template v-slot:header>
              <div class="v360-header text-light bg-dark">
                  <span class="v360-header-title">36 Images - Autoplay (1 loop) - Reverse Spin</span>
                  <span class="v360-header-description"></span>
              </div>
            </template>

          </vue-three-sixty>
        </div>
        <!--/ Example 1 - Car -->
      </div>
    </div>
  </div>
</template>

<script>

//import I360Viewer from './components/I360Viewer.vue'

export default {
  name: 'app',
  data: () => ({
    layers: [
      {
        imagePath: "http://localhost:5000/Pirmas Namas/Namo Spalva 1/Classic C/RR22 GREY",
        fileName: "0{index}.jpg",
        fileNamePadding: 3,
      },
      {
        imagePath: "http://localhost:5000/Fonas",
        fileName: "0{index}.png",
        fileNamePadding: 3,
      },
    ]
  }),
  mounted() {
    //safari zoom bug fix
    this.disableZoomin();
  },
  methods: {
    disableZoomin(){
      document.addEventListener("gesturestart", function (e) {
        e.preventDefault();
          document.body.style.zoom = 0.99;
      });
      document.addEventListener("gesturechange", function (e) {
        e.preventDefault();
        document.body.style.zoom = 0.99;
      });
      
      document.addEventListener("gestureend", function (e) {
          e.preventDefault();
          document.body.style.zoom = 1;
      });
    }
  }
}
</script>