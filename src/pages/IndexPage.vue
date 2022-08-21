<template>
    <!-- <div>
      <img
        alt="Quasar logo"
        src="~assets/quasar-logo-vertical.svg"
        style="width: 200px; height: 200px"
      />
    </div> -->
    <link type="text/css" rel="stylesheet" href="//golden-layout.com/assets/css/goldenlayout-base.css" />
    <link type="text/css" rel="stylesheet" href="//golden-layout.com/assets/css/goldenlayout-dark-theme.css" />
    <div ref ="goldenlayoutContainer">
      
    </div>
  
</template>

<script>
import { ref, onMounted, onUnmounted, onUpdated, createApp } from "vue";
import { GoldenLayout } from "golden-layout/src/index";
import TestCom from "src/components/TestCom.vue";

 const config = {
    root: {
        type: 'row',
        content: [
            {
                type: 'stack',
                content : [
                    {
                        title: 'others',
                        type: 'component',
                        componentType: 'TestCom',
                        width: 60,
                        height: 10,
                        componentState: { text: 'others' }
                    },
                    {
                        title: 'my video',
                        type: 'component',
                        componentType: 'me',
                        width: 60,
                        componentState: { text: 'my video' }
                    }
                ]
            },
            
            {
                type : 'column',
                content : [
                    {
                        title: 'chat',
                        type: 'component',
                        componentType: 'others',
                        // componentState: { text: 'Component 2' }
                    },
                    {
                        title: 'Teacher Code',
                        type: 'component',
                        componentType: 'me',
                        width: 40,
                    },
                ],
                width: 30
            }
        ]
    }
};
export default {
  name: 'IndexPage',

  setup(props) {

    /**
     * 특정 ID를 가진 html element가 생성될때까지 기다렸다가 콜백함수 실행.
     * @param {*} selector html element id
     * @param {*} callback 해당 객체가 생성되면 호출될 콜백 함수
     * @param {*} maxtries 최대 시도 횟수
     * @param {*} interval 체크 간격
     */
    const waitForEl = (selector, callback, maxtries = false, interval = 1000)=>{
        const poller = setInterval(() => {
            const el = document.getElementById(selector);
            const retry = maxtries === false || maxtries-- > 0
            if (retry && el.length < 1) return // will try again
            clearInterval(poller)
            callback(el || null)
        }, interval);
    }

    const goldenlayoutContainer = ref(undefined);
    let goldenLayout;
    
    // html element가 안겹치게 하기 위한 id counter
    let id = ref(1);

    onMounted(() => {
      goldenLayout= new GoldenLayout(goldenlayoutContainer);
      goldenLayout.registerComponent('me', function(container){
              this.rootElement = container.element;
              this.rootElement.innerHTML = '<img src="src/assets/me.png" alt="" style="width:100% height:100%">';
              this.resizeWithContainerAutomatically = true;
          });

      goldenLayout.registerComponent('others', function(container){
          this.rootElement = container.element;
          this.rootElement.innerHTML = '<img src="src/assets/others.png" alt="">';
          this.resizeWithContainerAutomatically = true;
      });

      goldenLayout.registerComponent('TestCom', function(container){
          
        this.rootElement = container.element;
        this.rootElement.innerHTML = `<div id="id${id.value}" style="height:200px"></div>`;
        this.resizeWithContainerAutomatically = true;

        waitForEl(`id${id.value}`, (target)=>{
          const comp = createApp({
              extends : TestCom,
              props: ["code"],
          },{
              code : "import java.util."
          });
          comp.mount(`#${target.id}`);
        });
      });
      
      goldenLayout.init();
      goldenLayout.loadLayout(config);

      goldenLayout.on('stateChanged',function(some){
        console.log("state Changed!");
      });
    });

    onUpdated(() => {
      console.log("updated");
    });

    onUnmounted(() => {
        goldenLayout.destroy();
    });

    return {
      waitForEl,
      goldenlayoutContainer,
      goldenLayout,
      id
      
    }
    
  }
}
</script>

<style scoped>
@import "src/css/goldenlayout-base.css";
@import "src/css/goldenlayout-dark-theme.css";
</style>