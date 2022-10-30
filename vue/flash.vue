<template>
    <div>
        <div>
        <button @click="rf(null, $event)">Read RF Config</button>
        <button @click="config(null, $event)">Read Config</button>
        </div>
        <div v-html="display" class="display"></div>
    </div>
</template>

<script>
  module.exports = {

    data: ()=>{
      return {
        msg: 'world!',
        rfdata: null,
        display: '',
        configdata: null,
        status:'nothing going on',
      }
    },
    methods:{
        rf(cb){
            this.status += '<br/>reading rf config...';
            let url = window.device+'/api/flash/1e0000-1000';
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.rfdata = buffer; 
                    console.log('received '+buffer.byteLength);
                    this.status += '..got rf config...';
                    this.dump(buffer);
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },
        config(cb){
            this.status += '<br/>reading rf config...';
            let url = window.device+'/api/flash/1e1000-1000';
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.configdata = buffer; 
                    console.log('received '+buffer.byteLength);
                    this.status += '..got rf config...';
                    this.dump(buffer);
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },

        dump(buffer){
            this.display = '<pre>';
            const view = new Uint8Array(buffer);

            let linestart = '0000: ';
            let lineend = '';

            for (let i = 0; i < buffer.byteLength; i++){
                if (i && !(i % 32)){
                    this.display += (linestart + ' : '+lineend + '\n');
                    linestart = ('000'+i.toString(16)).slice(-4) +': ';
                    lineend = '';
                    this.display += '\n';
                }
                linestart += ('0'+view[i].toString(16)).slice(-2) + ' ';
                lineend += (view[i] < 0x20)? '.':String.fromCharCode(view[i]);
            }
            this.display += linestart + ' : '+lineend + '\n';
            this.display += '</pre>';
        }

    },
    mounted (){
        this.msg = 'fred';
        console.log('mounted ota');
    }
  }
//@ sourceURL=/vue/flash.vue
</script>

<style scoped>
.display pre{
    font-family: 'Courier New', Courier, monospace;
    font-size: 14px;
}
</style>
