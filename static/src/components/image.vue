<template>
    <div>
        <div class="blogcover">
        </div>
        <div class="form-group">
            <label for="title" class="control-label"><label for="title">标题</label></label>
            <input class="form-control" v-model="data.title" placeholder="Write a nice title" type="text">
        </div>
        <markdown-editor v-model="data.body"></markdown-editor/>
        <div class="action-btns">
            <button class="btn btn-warning" v-on:click="save()">保存</button>
            <div v-if="canPublish">
                <button v-if="this.data.published" class="btn btn-primary" v-on:click="publish(false)">取消发布</button>
                <button v-else class="btn btn-primary" v-on:click="publish(true)">发布</button>
            </div>
        </div>
    </div>
</template>
<script>
import axios from 'axios'
import toastr from "toastr"
//import VueCoreImageUpload from 'vue-core-image-upload'
import { markdownEditor } from 'vue-simplemde'
import Cropper from 'cropperjs'
import 'cropperjs/dist/cropper.min.css'

function createCropperVanilla(){
    var image = document.getElementById('cropit');
    console.log(image)
    var cropper = new Cropper(image, {
        aspectRatio: 16 / 9,
        movable: false,
        autoCrop: false,
    })
    return cropper
}

function cleanCropper(){
    var ctn = document.getElementById('cropit-ctn');
    var image = document.getElementById('cropit');
    image.classList.remove("preview");
    image.classList.remove("cropper-hidden");
    var child = document.getElementsByClassName("cropper-container")[0];
    ctn.removeChild(child)
}

export default {
    components: {
        //   'vue-core-image-upload': VueCoreImageUpload,
        'markdown-editor': markdownEditor,
    },
    data() {
        return {
            src: '',
            blogpost_id: null,
            cropper: null,
            project: null,
            owner: null,
            data: {
                project_id: null,
                title: '',
                body: '',
                published: false
            },
            file_name: null,
            short_name: '',
        }
    },
    created(){
        var url = window.location.href 
        var reg = /\/project\/(\w+)/g
        var groups = reg.exec(url)
        if (groups !== null) this.short_name = groups[1]
        var update = false
        if (url.indexOf('/update') !== -1) {
            var tmp = url.split('/')
            this.blogpost_id = tmp[(tmp.length -2)]
            url = '/api/blogpost/' + this.blogpost_id
            update = true
        }
        var options = {headers: {'Content-Type': 'application/json'}}
        var self = this
        this.update = update
        axios({
            method:'get',
            url:url,
            headers: {'Content-Type': 'application/json'},
            data: null
        }).then(function(response) {
            self.project = response.data.project
            self.owner = response.data.owner
            if (update) {
                self.data.project_id = response.data.project_id
                self.data.title = response.data.title
                self.data.body = response.data.body
                self.src = response.data.media_url
                self.file_name = null
                self.data.published = response.data.published
            } else {
                self.data.project_id = response.data.project.id
            }
        });

    },

    methods: {
        createCropper(){
            this.cropper = createCropperVanilla()
        },
        cropIt(){
            var self = this
            if (this.cropper === null)  this.cropper = createCropperVanilla()
            self.cropper.getCroppedCanvas();
            self.cropper.getCroppedCanvas({
                width: 160,
                height: 90,
                beforeDrawImage: function(canvas) {
                    const context = canvas.getContext('2d');

                    context.imageSmoothingEnabled = false;
                    context.imageSmoothingQuality = 'high';
                },
            });


            if (document.querySelector('input[type=file]').files.length > 0) {
                this.file_name = document.querySelector('input[type=file]').files[0].name.split(".")[0] + ".png"
            }

            // Upload cropped image to server if the browser supports `HTMLCanvasElement.toBlob`
            self.cropper.getCroppedCanvas().toBlob(function (blob) {
                var formData = new FormData();

                formData.append('file', blob, self.file_name)
                formData.append('project_id', self.data.project_id)
                formData.append('title', self.data.title)
                formData.append('body', self.data.body)

                if (self.puturl === '/api/blogpost') {
                    axios.post(self.puturl, formData).then(function(response){
                        self.data.title = response.data.title
                        self.data.body = response.data.body
                        self.blogpost_id = response.data.id
                        self.src = response.data.media_url + '?' + Date.now()
                        self.cropper.destroy()
                        self.cropper = null
                    }
                    )
                } else {
                    axios.put(self.puturl, formData).then(function(response){
                        self.data.title = response.data.title
                        self.data.body = response.data.body
                        self.blogpost_id = response.data.id
                        self.src = response.data.media_url + '?' + Date.now()
                        self.cropper.destroy()
                        self.cropper = null
                    })
                }
            });

        },
        previewImage: function(event) {
            // Reference to the DOM input element
            var input = event.target;
            // Ensure that you have a file before attempting to read it
            if (input.files && input.files[0]) {
                // create a new FileReader to read this image and convert to base64 format
                var reader = new FileReader();
                // Define a callback function to run, when FileReader finishes its job
                reader.onload = (e) => {
                    // Note: arrow function used here, so that "this.imageData" refers to the imageData of Vue component
                    // Read image as base64 and set to imageData
                    this.src = e.target.result;
                }
                // Start the reader job - read file as a data url (base64 format)
                reader.readAsDataURL(input.files[0]);
            }
        },

        imageuploaded(res) {
            if(res.media_url !== '' && res.media_url !== undefined) {
                this.src = res.media_url
            }
            this.data.title = res.title
            this.data.body = res.body
            this.blogpost_id = res.id
        },
        save(){
            var self = this
            if (!this.update) {
                axios.post(this.puturl, this.data).then(function (response) {
                    if (response.status === 200) {
                        toastr.success('保存成功，可以发布了:D')
                    } else {
                        toastr.error("服务器开小差了:-(")
                    }
                    // toastr.clear()
                    if (response.data.media_url !== '' && response.data.media_url !== null) {
                        self.src = response.data.media_url
                    }
                    self.data.title = response.data.title
                    self.data.body = response.data.body
                    self.blogpost_id = response.data.id
                    self.update = true
                })
            } else {
                axios.put(this.puturl, this.data).then(function (response) {
                    if (response.status === 200) {
                        toastr.success('更新成功:D')
                    } else {
                        toastr.error("服务器开小差了:-(")
                    }
                    // toastr.clear()
                    setTimeout(function () {
                        window.location.href = "/project/" + self.short_name + "/blogs"
                    }, 3000)
                })
            }
        },
        publish(flag){
            this.data.published = flag
            var self = this
            if (!this.update) {
                axios.post(this.puturl, this.data).then(function (response) {
                    if (response.status === 200) {
                        toastr.success('更新成功:D')
                    } else {
                        toastr.error("服务器开小差了:-(")
                    }
                    if (response.data.media_url !== '' && response.data.media_url !== null) {
                        self.src = response.data.media_url
                    }
                    self.data.title = response.data.title
                    self.data.body = response.data.body
                    self.blogpost_id = response.data.id

                })
            } else {
                axios.put(this.puturl, this.data).then(function (response) {
                    if (response.status === 200) {
                        toastr.success('更新成功:D')
                    } else {
                        toastr.error("服务器开小差了:-(")
                    }
                })
            }
            // toastr.clear()
            var short_name = this.short_name
            setTimeout(function () {
                window.location.href = "/project/" + self.short_name + "/blogs"
            }, 3000)
        }
    },
    computed: {
        isCropping(){
            if (this.cropper !== null) return true
            else return false
        },
        puturl(){
            if (this.blogpost_id) return '/api/blogpost/' + this.blogpost_id
            else return '/api/blogpost'
        },
        canPublish(){
            if (this.data.title === '' || this.data.body === '') return false
            else return true
        }
    }
}
</script>
<style>
.blogcover {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.g-core-image-upload-btn {
    position: absolute !important;
}

.action-btns {
    display: flex;
    justify-content: space-between;
}

#cropit {
    max-width: 100%;
}
.cropit-ctn {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
}
</style>
