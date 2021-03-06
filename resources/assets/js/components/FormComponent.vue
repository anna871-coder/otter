<template>
    <div>
        <form>
            <div class="card">
                <div class="card-header">
                    <h3 class="card-title">{{ handleText }} {{ resourceName | beautify }}</h3>
                </div>
                <div class="card-body">
                    <div :class="['dimmer', (this.loading ? 'active' : '')]">
                        <div class="loader"></div>
                        <div class="dimmer-content">
                            <div class="row">
                                <div class="col-md-12">
                                    <div v-for="fieldType, fieldKey in resourceFields" class="form-group">
                                        <label class="form-label">{{ fieldKey | beautify }}</label>
                                        <textarea
                                                v-if="fieldType === 'textarea' || fieldType === 'wysiwyg'"
                                                :class="['form-control', (fieldType === 'wysiwyg' ? 'trumbowyg-textarea' : ''),(errors.first(fieldKey) ? 'is-invalid' : ''), (fields[fieldKey] && fields[fieldKey].dirty && !errors.first(fieldKey) ? 'is-valid' : '')]"
                                                v-model="resourceData[`${fieldKey}`]"
                                                v-bind:name="fieldKey"
                                                :data-vv-as="fieldKey | beautify"
                                                v-validate="(validationFields ? validationFields[fieldKey] : '')"
                                        >
                                            {{ (fieldType === 'wysiwyg') ? initTrumbowyg(fieldKey, resourceData[`${fieldKey}`]) : resourceData[`${fieldKey}`] }}
                                        </textarea>
                                        <input  v-else
                                                :class="['form-control', (errors.first(fieldKey) ? 'is-invalid' : ''), (fields[fieldKey] && fields[fieldKey].dirty && !errors.first(fieldKey) ? 'is-valid' : '')]"
                                                v-model="resourceData[`${fieldKey}`]"
                                                v-bind:name="fieldKey"
                                                :data-vv-as="fieldKey | beautify"
                                                v-bind:type="fieldType"
                                                v-validate="(validationFields ? validationFields[fieldKey] : '')">
                                        <div class="invalid-feedback">{{ errors.first(fieldKey) }}</div>
                                    </div>
                                    <div v-for="relationalMetaData, relationalKey in relationalFields" v-if="relationalData && relationalMetaData.relationshipType === 'BelongsTo' || relationalMetaData.relationshipType === 'BelongsToMany' || relationalMetaData.relationshipType === 'HasOne' || relationalMetaData.relationshipType === 'HasMany'" class="form-group">
                                        <label class="form-label">{{ relationalKey | beautify }}</label>
                                        <select class="form-control" multiple="true" v-if="relationalData && relationalMetaData.relationshipType === 'BelongsToMany' || relationalMetaData.relationshipType === 'HasMany'" v-model="relationalMetaData.relationshipId">
                                            <option disabled selected value="">Select {{ relationalKey | beautify }}</option>
                                            <option v-if="relationalData" v-for="option in relationalData[`${relationalKey}`]" v-bind:value="option.id">{{ option[`${relationalMetaData.resourceTitle}`] }}</option>
                                        </select>
                                        <p v-if="relationalMetaData.relationshipType === 'HasMany'" style="color: red;">*Removing data from a HasMany relationship will delete the record</p>
                                        <select class="form-control" v-if="relationalData && relationalMetaData.relationshipType === 'BelongsTo' || relationalMetaData.relationshipType === 'HasOne'" v-model="relationalMetaData.relationshipId">
                                            <option disabled selected value="">Select {{ relationalKey | beautify }}</option>
                                            <option v-if="relationalData" v-for="option in relationalData[`${relationalKey}`]" v-bind:value="option.id">{{ option[`${relationalMetaData.resourceTitle}`] }}</option>
                                        </select>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="card-footer text-right">
                    <alert-component
                            v-for="alert, alertIndex in alertData"
                            :key="alertIndex"
                            :alertLevel="alert.level"
                            :alertMessage="alert.message"
                    ></alert-component>
                    <button type="button" :class="['btn', 'btn-primary', 'btn-block', 'btn-black', (this.loading ? 'btn-loading' : '')]" @click="handleAction(action)">{{ handleButtonText }}</button>
                </div>
            </div>
        </form>
    </div>
</template>

<script>
    export default {
        name: "FormComponent",
        props: [
            'resourceId',
            'resourceName',
            'resourceFields',
            'validationFields',
            'relationalFields',
            'singularResourceName',
            'action',
            'pathPrefix'
        ],
        data() {
            return {
                handleText: 'Create',
                handleButtonText: 'Create',
                loading: false,
                selectizeEventFired: false,
                alertData: [],
                resourceData: {},
                relationalData: {},
                trumbowygData: {},
            }
        },
        created() {
            if(this.action === 'update')
            {
                this.handleText = "Edit";
                this.handleButtonText = "Update";
                this.fetchResource();
            }
            else if(this.action === 'create')
            {
                this.handleText = "Create";
                this.handleButtonText = "Create";
            }
            this.fetchRelationalItems();
        },
        mounted() {
        },
        updated() {
        },
        methods: {
            initTrumbowyg(name, value) {
                $('.trumbowyg-textarea').trumbowyg({
                    svgPath: '/assets/fonts/trumbowygicons.svg',
                    removeformatPasted: true,
                    resetCss: true,
                    autogrow: true
                })
                    .on('tbwchange', (event) => {
                        if(event.target.value !== this.resourceData[`${name}`])
                        {
                            this.setTrumbowygValue(name, event.target.value);
                        }
                    });
                $(`textarea[name='${name}'].trumbowyg-textarea`).trumbowyg('html', value);
            },
            setTrumbowygValue(fieldKey, value) {
                if(value !== this.trumbowygData[fieldKey])
                {
                    this.trumbowygData[fieldKey] = value;
                }
            },
            getTrumbowygValue(fieldKey) {
                return this.trumbowygData[fieldKey];
            },
            fetchResource() {
                axios.get(`/api/${this.pathPrefix}/${this.resourceName}/${this.resourceId}`)
                    .then(response=>{
                        this.resourceData = response.data.data;
                    })
                    .catch(e => {
                        this.error = `Could not retrieve ${this.resourceName}. Server error.`;
                    })
                    .finally(() => {
                    });
            },
            fetchRelationalItems() {
                axios.get(`/api/${this.pathPrefix}/${this.resourceName}/relational`)
                    .then(response=>{
                        this.relationalData = response.data.data;
                    }).catch(e => {
                        this.error = `Could not retrieve ${this.resourceName}. Server error.`;
                    })
                    .finally(() => {
                        let self = this;
                        $('select')
                            .selectize({})
                            .on('change', function(e) {
                                if(!self.selectizeEventFired)
                                {
                                    self.selectizeEventFired = true;
                                    let event = new Event('change');
                                    e.target.dispatchEvent(event);
                                }
                                else
                                {
                                    self.selectizeEventFired = false;
                                }
                            });
                    });
            },
            handleAction(action) {
                this.$validator.validate().then(valid => {
                    if (valid) {
                        this.loading = true;
                        if(action === 'update')
                        {
                            this.handleUpdate();
                        }
                        else if(action === 'create')
                        {
                            this.handleStore();
                        }
                    }
                });
            },
            handleStore(e) {
                this.resourceData.relationalFields = this.relationalFields;

                axios.post(`/api/${this.pathPrefix}/${this.resourceName}`, this.resourceData)
                    .then(response => {
                        console.log("success");
                        // window.location = response.data.redirect;
                        window.location = `/${this.pathPrefix}/${this.resourceName}`;
                    })
                    .catch(e => {
                        this.alertData.push({
                            "level" : "danger",
                            "message" : this.$options.filters.beautify(this.singularResourceName) + " could not be saved. Please check your values or try again later."
                        });
                    })
                    .finally(() => {
                        this.loading = false;
                    });
            },
            handleUpdate(e) {
                Object.keys(this.resourceFields).forEach(fieldKey => {
                    if (this.resourceFields[fieldKey] === 'wysiwyg') {
                        this.resourceData[`${fieldKey}`] = this.getTrumbowygValue(fieldKey)
                    }
                })

                this.resourceData.relationalFields = this.relationalFields;

                axios.patch(`/api/${this.pathPrefix}/${this.resourceName}/${this.resourceId}`, this.resourceData)
                    .then(response => {
                        this.fetchResource();
                        this.alertData.push({
                            "level" : "success",
                            "message" : this.$options.filters.beautify(this.singularResourceName) + " has been successfully updated"
                        });
                    })
                    .catch(e => {
                        this.alertData.push({
                            "level" : "danger",
                            "message" : this.$options.filters.beautify(this.singularResourceName) + " could not be updated. Please check your values or try again later."
                        });
                    })
                    .finally(() => {
                        this.loading = false;
                    });
            },
        }
    }
</script>