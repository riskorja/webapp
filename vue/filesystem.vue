<template>
    <div class="fill">
        <p>"List Filesystem" will display all files on LittleFS. 
        After listing the files, you can click on file name to begin editing text.
        "Create File" allows you to create a new file on Device (like autoexec.bat).
        You can also add files by drag &amp; dropping it.
        "Reset scripts" will stop all script threads running (if you have started any) without stopping the device.
        When editing a file, use "Save, Reset (...), and run" file to test new scripts, it will restart scripting every time with a clear state.
        </p>
        <div class="top">
            <button @click="backup(null, $event)">Read fsblock</button>
            <button @click="restore(null, $event)">Restore fsblock</button>
            <button @click="read(null, $event)">List Filesystem</button>
            <button @click="create(null, $event)">Create File</button>
            <button @click="resetSVM(null, $event)">Reset scripts</button>
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
                <h2 id="fileEditorLabel">File editor. Select file to begin.</h2>
                <div id="fileEditorBody" style="display:none">
                    <button @click="save(null, $event)">Save</button>
                    <button @click="save(startScript_simple)">Save, Run file as script thread</button>
                    <button @click="save(startScript_firstReset)">Save, Reset SVM and run file as script thread</button>         
                    <button @click="deleteFile(null, $event)">Delete</button>
                    <textarea v-model="edittext" rows="40" cols="100" style="height:90%"></textarea>
                </div>
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
            let readCallback = this.read;
            
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
                    readCallback();
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
            if(fname == null)
            {
                 alert("Canceled.");
            }
            else if(fname.length < 1)
            {
                 alert("Empty name.");
            }
            else
            {
                 //alert("TODO "+fname);
                 alert("Will try to create " +fname);
                 this.savefile(fname,"");
            }
        },
        
        editfile(name){
            let url = window.device+'/api/lfs'+name;
            fetch(url)
                .then(response => response.text())
                .then(text => {
                    this.edittext = text;
                    this.editname = name;
                    document.getElementById("fileEditorLabel").innerHTML = "Editing "+name;
                    document.getElementById("fileEditorBody").style.display = "block";
                });
        },


        deleteFile(cb) {   
            let readCallback = this.read;
            if (this.editname) {
                let r = confirm("Do you really want to remove the file "+this.editname+"?");
                if(r == false)
                {
                     alert("Ok, then not.");
                     return;
                }
                let url = window.device+'/api/del'+this.editname;
                alert("Will try to remove - url is "+url);
                fetch(url)
                    .then(response => response.arrayBuffer())
                    .then(buffer => {
                        this.status += '..delete done...';
                         if (cb) cb();
                         readCallback();
                    })
            } else {
                alert("Please begin editing some file first. Just click the name on list to edit.");
            }
        },
        save(cb) {
            let readCallback = this.read;
            if (this.editname) {
                let url = window.device+'/api/lfs'+this.editname;
                fetch(url, { 
                        body: this.edittext,
                        method: 'POST',
                    })
                    .then(()=>{
                         if (cb) cb();
                         readCallback();
                    });
            } else {
                alert("Please begin editing some file first. Just click the name on list to edit.");
            }
        },
        resetSVM() {
            if (this.editname) {
                let url = window.device+'/api/cmnd';
                let cmd = "";
                
                cmd = "backlog resetSVM; ";
                    
                fetch(url, { 
                        body: cmd,
                        method: 'POST',
                    })
                    .then(()=>{
                         
                    });
            }
        },
        startScript_simple() {
            this.startScript(null,null,0);
        },
        startScript_firstReset() {
            this.startScript(null,null,1);
        },
        startScript(cb, event, bResetAll) {
            if (this.editname) {
                let url = window.device+'/api/cmnd';
                let cmd = "";
                if(bResetAll == 1)
                {
                    cmd = "backlog resetSVM; ";
                }
                cmd += "startScript " + this.editname;
                fetch(url, { 
                        body: cmd,
                        method: 'POST',
                    })
                    .then(()=>{
                         
                    });
            } else {
                alert("Please begin editing some file first. Just click the name on list to edit.");
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
        position: relative;
    }
</style>
