<template>
    <div>
        <div>
            <p>Chipset: {{chipset}}</p>
            <p>Build: {{build}}</p>
        </div>
        <div>
            <button :disabled="!otadata" @click="sequence(null, $event)">Start safe OTA (keep LittleFS data)</button>
            <br/>
            <button :disabled="!otadata" @click="startota(null, $event)">Start quick OTA (delete all LittleFS data)</button>
            <br/>
            <br/>
            <button @click="backup(null, $event)">Read fsblock</button>
            <button @click="reboot(null, $event)">Reboot</button>
            <button @click="restore(null, $event)">Restore fsblock</button>
            <select v-model="defaultaction">
                <option value=''>Do nothing</option>
                <option value='ota'>OTA Only</option>
                <option value='sequence'>Backup/OTA/Restore</option>
            </select>
            <span>Selected: {{ defaultaction }}</span>
        </div>
        <br/>
        <div>
            <label for="otaFilePicker">Select OTA (RBL) file:</label><input id="otaFilePicker" type="file" @change="fileSelected($event)" :accept="otaFileExtension">
            <div class="drop" @drop="dropHandler($event)" @dragover="dragOverHandler($event)">
                <div class="otatext center" v-html="otatext"></div>
            </div>
            <div v-html="status" :class="{invalid: invalidOTASelected}"></div>
        </div>
    </div>
</template>

<script>
  module.exports = {

    data: ()=>{
      return {
        msg: 'world!',
        backupdata: null,
        otadata:null,
        otatext:'Drop OTA file here',
        status:'Nothing going on',
        defaultaction: '',
        build:'unknown',
        chipset:'unknown',
        invalidOTASelected: false,
        otaFileExtension:".rbl,.img"
      }
    },
    methods:{
        getinfo(){
            let url = window.device+'/api/info';
            fetch(url)
                .then(response => response.json())
                .then(res => {
                    this.build = res.build;
                    this.chipset = res.chipset;     //Set chipset to fixed value for testing

                    if (this.chipset){
                        this.otaFileExtension = this.chipSetUsesRBL() ? ".rbl" : ".img";
                    }
                })
                .catch(err => {
                    this.error = err.toString();
                    console.error(err)
                }); // Never forget the final catch!
        },

        /* Check if the ArrayBuffer represents RBL file */
        isRBL(arrayBuffer){
            let view = new DataView(arrayBuffer);
            if (view.byteLength < 3) return false;
            console.log(view);
            return view.getUint8(0) === 82 && view.getUint8(1) === 66 && view.getUint8(2) === 76;
        },

        /* Check if the chipset uses RBL files */
        chipSetUsesRBL(){
            return this.chipset === "BK7231T" || this.chipset === "BK7231N";
        },

        /* Check if the ota fileName matches the chipset */
        fileNameMatchesChipset(fileName){
            if (!this.chipset){     //Accept any file if chipset missing (older firmware)
                return true; 
            }

            //e.g. OpenW800_1.12.40_ota.img, OpenBK7231N_1.12.40.rbl, OpenW800_1.12.40_ota.img
            var lowerName = fileName.toLowerCase();
            if (!lowerName.startsWith("open" + this.chipset.toLowerCase() + "_")) return false;

            var ext = this.chipSetUsesRBL() ? ".rbl" : ".img";
            return lowerName.endsWith(ext);
        },

        /* Check ota data from file selection/drop */
        checkOTAData(event, file, operation){
            this.otadata = null;    //Reset otadata
            
            var result = event.target.result;   //ArrayBuffer
            console.log("Checking ota data");
            console.log(result);
            console.log('otadata len:' + result.byteLength);
            this.otatext = file.name + ' len:' + result.byteLength;

            if (this.chipSetUsesRBL()){
                if (!this.isRBL(result)){   //if the file is not RBl then it is right away invalid
                    this.invalidOTASelected = true;
                }
                else{
                    //Prevent BK7231N from being used in BK7231T
                    this.invalidOTASelected = !this.fileNameMatchesChipset(file.name);
                }
            }
            else{
                //At this point W800 is the only other chipset with has OTA images e.g. OpenW800_1.12.40_ota.img
                //img file doesn't seem to have any marker bytes so we will just check the file name
                this.invalidOTASelected = !this.fileNameMatchesChipset(file.name);
            }

            if (this.invalidOTASelected){
                this.status = 'Invalid OTA file was ' + operation;
            }
            else{
                this.status = 'OTA file '+ operation;
                this.otadata = result;
            }
        },
        fileSelected(ev){
            console.log("File selected");
            this.invalidOTASelected = false; //Reset status style

            var file = ev.target.files[0];  //There should be only one file
            if (file){
                console.log('... fileName = ' + file.name);
                var reader = new FileReader();
                reader.onload = (event) => this.checkOTAData(event, file, "selected");
                reader.readAsArrayBuffer(file);
            }
        },
        dropHandler(ev){
            ev.preventDefault();
            if (ev.dataTransfer.items) {
                console.log('Dropped ' + ev.dataTransfer.items.length + ' items');

                // Use DataTransferItemList interface to access the file(s)
                for (var i = 0; i < ev.dataTransfer.items.length; i++) {
                    // If dropped items aren't files, reject them
                    if (ev.dataTransfer.items[i].kind === 'file') {
                        var file = ev.dataTransfer.items[i].getAsFile();
                        console.log('... file[' + i + '].name = ' + file.name);
                        var reader = new FileReader();
                        reader.onload = (event) => this.checkOTAData(event, file, "dropped");
                        
                        console.log(file);
                        reader.readAsArrayBuffer(file);
                    }
                }
            }
        },

        dragOverHandler(ev){
            //console.log('File(s) in drop zone');
            // Prevent default behavior (Prevent file from being opened)
            ev.preventDefault();
        },

        sequence(){
            if (this.otadata){
                this.backup(()=>{
                    this.startota(()=>{
                        this.status += '<br/>waiting for reboot...';
                        let count = 40;
                        let checkrestore = ()=>{
                            count--;
                            if (!count){
                                this.restore(()=>{
                                    this.status += '<br/>Sequence complete';
                                });
                            } else {
                                this.status += '.'+count;
                                setTimeout(checkrestore, 1000);
                            }
                        };
                        checkrestore();
                    })
                })
            } else {
                this.status = 'Drop an OTA file first...';
            }
        },

        reboot(cb){
            let url = window.device+'/api/reboot';
            fetch(url, {
                method: 'POST',
                body: ''
            })
                .then(response => response.text())
                .then(text => {
                    if(cb) cb();
                });
        },

        backup(cb){
            this.status += '<br/>starting backup...';
            let url = window.device+'/api/fsblock';
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.backupdata = buffer; 
                    console.log('received '+buffer.byteLength);
                    this.status += '..backup done...';
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },

        startota(cb){
            this.status += '<br/>starting OTA...';
            console.log('start ota ');
            let url = window.device+'/api/ota';
            if (this.otadata){
                fetch(url, { 
                        method: 'POST',
                        body: this.otadata
                    })
                    .then(response => response.text())
                    .then(text => {
                        console.log('received '+text);
                        this.otatext += ' finished:'+text;
                        this.status += '<br/>rebooting...';
                        this.reboot(cb);
                    })
                    .catch(err => console.error(err)); // Never forget the final catch!
            }
        },
        restore(cb){
            this.status += '<br/>starting restore...';
            let url = window.device+'/api/fsblock';
            if (this.backupdata){
                fetch(url, { 
                        method: 'POST',
                        body: this.backupdata
                    })
                    .then(response => response.text())
                    .then(text => {
                        console.log('received '+text);
                        this.status += 'Restore complete...';
                        if(cb) cb();
                    })
                    .catch(err => console.error(err)); // Never forget the final catch!
            }
        },
    },
    mounted (){
        this.msg = 'fred';
        this.getinfo();
        console.log('mounted ota');
    }
  }
//@ sourceURL=/vue/controller.vue
</script>

<style scoped>
    .drop {
        border: 5px solid blue;
        width:  200px;
        height: 100px;
        text-align: center;
        position: relative;
        vertical-align: center;
    }

    .otatext {
    }
    .invalid{
        font-weight: bold;
        color: red;
    }
    .center {
        margin: 0;
        position: absolute;
        top: 50%;
        left: 50%;
        -ms-transform: translate(-50%, -50%);
        transform: translate(-50%, -50%);
    }
</style>
