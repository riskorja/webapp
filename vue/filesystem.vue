<template>
    <div class="fill">
        <div class="top">
            <button @click="backup(null, $event)">Read fsblock</button>
            <button @click="restore(null, $event)">Restore fsblock</button>
            <button @click="read(null, $event)">List Filesystem</button>>
            <button @click="create(null, $event)">Create File</button>
        </div>
        <div class="bottom">
            <div class="left">
                <input type="text" v-model="folder">
                <div class="drop" @drop="dropHandler($event)" @dragover="dragOverHandler($event)">
                    <div class="otatext center" v-html="otatext"></div>
                </div>
                <div v-html="status"></div>
                <div v-html="output"></div>
            </div>
            <div class="middle">
                <div>
                    <div v-for="file in files" v-bind:key="file.name">
                        <span v-if="file.type === 1"><button @click="editfile(file.name)">{{file.name}}</button> - {{file.size}}</span>
                        <br v-if="file.type === 1"/>
                    </div>
                </div>
            </div>
            <div class="right">
                <button @click="save(null, $event)">Save</button>
                <textarea v-model="edittext" rows="40" cols="100" style="height:90%"></textarea>
            </div>
        </div>
    </div>
</template>

<script>
  module.exports = {

    data: ()=>{
      return {
        msg: 'world!',
        backupdata: null,
        status:'nothing going on',
        folder:'',
        otatext:'drop file(s) here',

        output: '',

        edittext:'',
        editname:'',

        stack:[],
        files:[],
      }
    },
    methods:{
        dropHandler(ev){
            ev.preventDefault();
            console.log('drop');
            if (ev.dataTransfer.items) {
                this.getFilesDataTransferItems(ev.dataTransfer.items)
                .then(files => {
                    console.log(files);
                    let allfiles = [];
                    let addfolder = (f)=>{
                        for (let i = 0; i < f.length; i++){
                            if (f[i].subfolder){
                                addfolder(f[i].subfolder);
                            } else {
                                allfiles.push(f[i]);
                            }
                        }
                    };

                    addfolder(files);

                    console.log(allfiles);

                    this.uploadfiles(allfiles);

                });
                return;
            }
        },



        uploadfiles(files, cb) {
            let filecount = 0;

            let saveone = ()=>{
                if (filecount < files.length){
                    reader = new FileReader();
                    reader.onload = (event) => {
                        console.log(event);
                        console.log('file len:'+event.target.result.byteLength);
                        this.status += '<br/>save '+files[filecount].filepath;
                        this.savefile(files[filecount].filepath, event.target.result, ()=>{
                            filecount++;
                            this.status += '.. done';
                            saveone();
                        });
                        //holder.style.background = 'url(' + event.target.result + ') no-repeat center';
                    };
                    reader.readAsArrayBuffer(files[filecount]);
                } else {
                    if (cb) cb();
                }
            }

            saveone();
        },

        getFilesDataTransferItems(dataTransferItems) {
            function traverseFileTreePromise(item, path = "", folder) {
                return new Promise(resolve => {
                    if (item.isFile) {
                        item.file(file => {
                            file.filepath = (path || "") + file.name; //save full path
                            folder.push(file);
                            resolve(file);
                        });
                    } else if (item.isDirectory) {
                        let dirReader = item.createReader();
                        dirReader.readEntries(entries => {
                        let entriesPromises = [];
                        subfolder = [];
                        folder.push({ name: item.name, subfolder: subfolder });
                        for (let entr of entries)
                            entriesPromises.push(
                            traverseFileTreePromise(entr, (path || "")  + item.name + "/", subfolder)
                            );
                        resolve(Promise.all(entriesPromises));
                        });
                    }
                });
            }

            let files = [];
            return new Promise((resolve, reject) => {
                let entriesPromises = [];
                for (let it of dataTransferItems)
                entriesPromises.push(
                    traverseFileTreePromise(it.webkitGetAsEntry(), null, files)
                );
                Promise.all(entriesPromises).then(entries => {
                resolve(files);
                });
            });
        },



        dragOverHandler(ev){
            //console.log('File(s) in drop zone');
            // Prevent default behavior (Prevent file from being opened)
            ev.preventDefault();
        },

        savefile(name, data, cb){
            this.status += '<br/>saving file...';
            let url = window.device+'/api/lfs/';
            if (this.folder){
                url += this.folder;
                url += '/'
            }
            url += name;
            fetch(url, { 
                    method: 'POST',
                    body: data
                })
                .then(response => response.text())
                .then(text => {
                    console.log('received '+text);
                    this.status += 'save complete...';
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
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

        readFolder(fpath){
            let url = window.device+'/api/lfs'+fpath;
            fetch(url)
                .then(response => response.json())
                .then(folder => {
                    console.log('folder '+url,folder);
                    this.status += 'Read '+url;
                    if (!folder.content) return;
                    for (let i = 0;i < folder.content.length; i++){
                        if (folder.content[i].name.startsWith('.')){
                        } else {
                            let root = fpath;
                            if (fpath === '/'){
                                fpath = '';
                            }
                            this.files.push({ name:fpath + '/' + folder.content[i].name, 
                                size:folder.content[i].size,
                                type:folder.content[i].type});
                        }
                    }

                    for (let i = 0;i < this.files.length; i++){
                        if (this.files[i].type === 2 && !this.files[i].requested) {
                            this.readFolder(this.files[i].name);
                            this.files[i].requested = true;
                            break;
                        }
                    }
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },

        read(cb) {
            this.files = [];
            this.readFolder('/');
        },


        create(cb) {
            let fname = prompt("Please enter new file name:", "autoexec.bat");
        
        
            alert("TODO "+fname);
        },
        
        editfile(name){
            let url = window.device+'/api/lfs'+name;
            fetch(url)
                .then(response => response.text())
                .then(text => {
                    this.edittext = text;
                    this.editname = name;
                });
        },

        save() {
            if (this.editname) {
                let url = window.device+'/api/lfs'+this.editname;
                fetch(url, { 
                        body: this.edittext,
                        method: 'POST',
                    })
                    .then(()=>{

                    });
            }
        }

    },
    mounted (){
        this.msg = 'fred';
        console.log('mounted ota');
    }
  }
//@ sourceURL=/vue/filesystem.vue
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
    .center {
        margin: 0;
        position: absolute;
        top: 50%;
        left: 50%;
        -ms-transform: translate(-50%, -50%);
        transform: translate(-50%, -50%);
    }

    .left {
        position: absolute;
        left:0;
        width:30%;
        height:100%;
    }
    .middle {
        position: absolute;
        left:30%;
        width:20%;
        height:100%;
    }
    .right {
        position: absolute;
        left:50%;
        width:50%;
        height:100%;
    }
    .fill {
        height:90%;
    }
    .top {
        height:10%;
    }
    .bottom {
        height:90%;
    }
</style>
