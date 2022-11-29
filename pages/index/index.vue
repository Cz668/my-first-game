<template>
  <view>
      <canvas id="collisionCtx" type="2d" v-show="canvasShow"></canvas>
      <canvas id="canvas1" type="2d" @touchstart="touch"></canvas>
      <button v-if="restore" @click="restoreGame">重新开始</button>
  </view>
</template>

<script>
  var canvas = {}
  var canvasCtx = {}
  var particles = []
  var gameOver = false
  var explosions = []
  var collisionCanvas = {}
  var collisionCtx = {}
  var ravens = []
  // var innerAudioContext = wx.createInnerAudioContext({
  //           useWebAudioImplement: true 
  //         })
  class Raven{
      constructor(){
          this.spritWidth = 271
          this.spritHeight = 194
          this.sizeModifier = Math.random()*0.6+0.4
          this.width = this.sizeModifier*this.spritWidth
          this.height = this.sizeModifier*this.spritHeight
          this.x = canvas.width//保证乌鸦的路径是窗口的全部
          this.y = Math.random()*(canvas.height-this.height) //防止乌鸦出现在窗口的下面或只出现一部分
          this.directionX = Math.random()*5+3 //水平速度
          this.directionY = Math.random()*5-2.5;//垂直速度 
          this.markForDeletion=false 
          this.image = canvas.createImage()
          this.image.src = '../../static/img/05/项目 5：傻瓜射击游戏.png'
          this.frame = 0
          this.maxFrame = 4
          this.timeSinceFlap = 0
          this.flapInterval = Math.random()*50+50  //数值越大扇动翅膀的速度越慢
          this.randomColor = [Math.floor(Math.random()*255),Math.floor(Math.random()*255),Math.floor(Math.random()*255)]
          this.color = `rgb(${this.randomColor[0]},${this.randomColor[1]},${this.randomColor[2]})`  //用于碰撞检验
          this.hasTrail = Math.random()>0.5
      }
      update(deltatime){
          if(this.y<0||this.y>canvas.height-this.height){ //防止乌鸦跳出画布高度
              this.directionY=this.directionY*-1
          }
          this.x -=this.directionX
          this.y +=this.directionY
          if(this.x<0-this.width) this.markForDeletion=true   //假如乌鸦达到屏幕外就可以删除
          this.timeSinceFlap +=deltatime
          if(this.timeSinceFlap>this.flapInterval){
              if(this.frame>this.maxFrame) this.frame=0
              else this.frame++
              this.timeSinceFlap=0
              if(this.hasTrail){
                  for(let i=0;i<5;i++){//限制粒子效果只出现5个
                      particles.push(new Particle(this.x,this.y,this.width,this.color))
                  }
              }
          }
          if(this.x<0-this.width) gameOver=true   //乌鸦碰到边界就判断游戏结束
      }
      draw(){
          collisionCtx.fillStyle=this.color
          collisionCtx.fillRect(this.x,this.y,this.width/2,this.height/2)
          canvasCtx.drawImage(this.image,this.spritWidth*this.frame,0,this.spritWidth,this.spritHeight,this.x,this.y,this.width/2,this.height/2)
          
      }
  }
  
  // 乌鸦身体后的粒子效果
  class Particle{
      constructor(x,y,size,color){
          this.size=size
          this.x = x+this.size/3
          this.y = y+this.size/4
          this.radius=Math.random()*this.size/10
          this.maxRadius=Math.random()*20+35
          this.markForDeletion=false
          this.speedX=Math.random()*1+0.5
          this.color=color
      }
      update(){
          this.x+this.speedX
          this.radius+=0.8
          if(this.radius>this.maxRadius) this.markForDeletion=true
      }
      draw(){
          canvasCtx.save()
          canvasCtx.globalAlpha=1-this.radius/this.maxRadius
          canvasCtx.beginPath()
          canvasCtx.fillStyle=this.color
          canvasCtx.arc(this.x,this.y,this.radius,0,Math.PI*2)
          canvasCtx.fill()
          canvasCtx.restore()
      }
  }
  
  class Explosion{
      constructor(x,y,size){
          this.image=canvas.createImage()
          this.image.src = '../../static/img/05/项目 5：尘云.png'
          this.spritWidth=200
          this.spritHeight=179
          this.size=size
          this.x=x
          this.y=y
          this.frame=0
          // innerAudioContext.src = '../../sound/cloud/Ice attack 2.wav'
          this.timeSinceLastFrame=0
          this.frameInterVal=200
          this.markForDeletion=false}
      update(deltatime){
        //音乐无法实现因为微信小程序只支持服务器，不支持本地音频
          // if(this.frame===0) {
          //   innerAudioContext.play() // 播放
          // }
          this.timeSinceLastFrame+=deltatime
          if(this.timeSinceLastFrame>this.frameInterVal){
              this.frame++
              this.timeSinceLastFrame=0
              if(this.frame>5) this.markForDeletion=true
          }
      }
      draw(){
          canvasCtx.drawImage(this.image,this.frame*this.spritWidth,0,this.spritWidth,this.spritHeight,this.x,this.y,this.size/2,this.size/2)
      }
  }
  export default {
    data() {
      return {
        canvasShow: false,
        frame: 0,
        maxFrame: 4,
        lastTime: 0,
        ravenInterval: 1000,
        timeToNextRaven: 0,
        rafNun: {},
        num: 0,
        score: 0,
        restore: false
      }
    },
    
    onReady() {
      this.getCanvas()
    },
    methods: {
      getCanvas() {
        wx.createSelectorQuery()
              .select('#canvas1')
              .fields({
                node: true,
                size: true,
              })
              .exec((res) => {
                  canvas = res[0].node
                  canvasCtx = canvas.getContext('2d')
                  canvasCtx.font='50px Impact'
                  canvas.width = uni.getWindowInfo().windowWidth
                  canvas.height = uni.getWindowInfo().windowHeight
                  
                })
            wx.createSelectorQuery()
              .select('#collisionCtx')
              .fields({
                node: true,
                size: true,
              })
              .exec((res) => {
                collisionCanvas = res[0].node
                collisionCtx = collisionCanvas.getContext('2d')
                collisionCanvas.width = uni.getWindowInfo().windowWidth
                collisionCanvas.height = uni.getWindowInfo().windowHeight
                this.animate(0)
              })
      },
      touch(e){
        const m = e.touches
        const detectPixelColor = collisionCtx.getImageData(m[0].x,m[0].y,1,1)   //获取鼠标点击的那块区域的相关属性
        const pc=detectPixelColor.data  //鼠标点击的那块区域的rgb值如rgb(0,0,0,0.1)
        // console.log('pc',pc)
        ravens.forEach(object=>{ 
          //对每个乌鸦进行碰撞检测
            if(object.randomColor[0]===pc[0] && object.randomColor[1]===pc[1] && object.randomColor[2]===pc[2]){    //假如鼠标点击区域每部分的颜色都和乌鸦的颜色一种
                object.markForDeletion=true //将乌鸦的状态改为可以删除
                this.score++ //加分
                explosions.push(new Explosion(object.x,object.y,object.width))  //生成爆炸类
            }
        })
      },
      animate(timestamp){
          canvasCtx.clearRect(0,0,canvas.width,canvas.height)
          collisionCtx.clearRect(0,0,canvas.width,canvas.height)
          let deltatime = timestamp - this.lastTime  //0-
          this.lastTime = timestamp
          this.timeToNextRaven +=deltatime
         if(this.timeToNextRaven > this.ravenInterval){
          ravens.push(new Raven())
          this.timeToNextRaven=0   //生成乌鸦后重新计时
          ravens.sort(function(a,b){  //大乌鸦在前面，小乌鸦在后面
              return a.width-b.width
          })
         }
         this.drawScore(); //左上的游戏分数面版
         [...particles,...ravens,...explosions].forEach(object=>{
           object.update(deltatime)
          });
         [...particles,...ravens,...explosions].forEach(object=>{object.draw()})
         ravens = ravens.filter(object=>!object.markForDeletion)
         explosions = explosions.filter(object=>!object.markForDeletion)
         particles = particles.filter(object=>!object.markForDeletion)
         if(!gameOver){
          canvas.requestAnimationFrame(this.animate)
         }else{
          this.drawGameOver() //游戏结束面板
          this.restore = true
         }
      },
      
      
      drawScore(){//绘制左上的游戏分数面版
          canvasCtx.fillStyle='black'
          canvasCtx.fillText('Score:'+this.score,50,75)  //阴影效果
          canvasCtx.fillStyle='white'
          canvasCtx.fillText('Score:'+this.score,50,76)
      },
      
      drawGameOver(){    //绘制游戏结束面板
          canvasCtx.textAlign='center'
          canvasCtx.fillStyle='black'
          canvasCtx.fillText('Game Over your score is :'+this.score,canvas.width/2,canvas.height/2)
          canvasCtx.fillStyle='white'
          canvasCtx.fillText('Game Over your score is :'+this.score,canvas.width/2,canvas.height/2+2)
      },
      
      restoreGame() {
        this.restore = false
        this.score = 0
        // console.log('restore',this.restore)
        gameOver = false
        ravens = []
        particles = []
        explosions = []
        this.getCanvas()
      }
    }
  }
</script>

<style lang="scss">

canvas{
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
}
</style>
