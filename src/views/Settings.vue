<template>
    <div class="settings">
        <h1>Settings</h1>
<!--        <h2>Custom API Key</h2>-->
<!--        <p class="caption">Fill in your own Google Maps JavaScript API key.-->
<!--            Since the main API key doesn't work anymore this is required to play. Get an API key-->
<!--            <a href="https://developers.google.com/maps/documentation/javascript/get-api-key">here</a>.-->
<!--        </p>-->
<!--        <v-text-field label="API Key" v-model="apiKey"/>-->
<!--        <div>-->
<!--            <v-btn @click="setApiKey">Apply API key</v-btn>-->
<!--            <v-btn @click="removeApiKey">Remove Custom API key</v-btn>-->
<!--        </div>-->
        <h2>Game</h2>
        <v-switch label="Show finding random location visual (might spoil location)" v-model="showVisual"/>
        <p class="caption">Slow CPU mode reduces CPU usage of the location finding algorithm while you're playing the
            game.
            This will make StreetView panoramas smoother, but finding locations will take more time.</p>
        <v-switch v-model="slowCpu" label="Slow CPU mode"/>
        <h2>Theme color</h2>
        <v-color-picker mode="hexa" v-model="color" hide-mode-switch/>
        <br>
        <v-btn @click="resetColor()">Reset Theme Color</v-btn>
        <img src="../assets/favicon256.png" v-show="false" ref="favicon">
        <canvas ref="canvas" v-show="false"/>
        <div v-if="$store.state.realAccount">
            <h2>Account</h2>
            <v-btn :loading="loadingLogout" @click="logout">Logout</v-btn>
            <br>
            <v-dialog v-model="pwDialog" width="500">
                <template v-slot:activator="{ on }">
                    <v-btn v-on="on">Change Password</v-btn>
                </template>
                <v-card :loading="loadingChangePassword">
                    <v-form @submit="updatePassword">
                        <v-card-title primary-title>Change Password</v-card-title>
                        <v-card-text>
                            <v-text-field disabled type="mail" name="user" :value="$store.getters.user.email"
                                          label="Email"/>
                            <v-text-field type="password" name="newPassword" v-model="password"
                                          label="New Password"/>
                            <p class="caption error--text">{{pwError}}</p>
                        </v-card-text>
                        <v-divider/>
                        <v-card-actions>
                            <v-spacer/>
                            <v-btn color="primary" text type="submit">Submit</v-btn>
                        </v-card-actions>
                    </v-form>
                </v-card>
            </v-dialog>
            <br>

            <v-dialog v-model="deleteDialog" width="500">
                <template v-slot:activator="{ on }">
                    <v-btn color="error" v-on="on">Delete Account</v-btn>
                </template>
                <v-card :loading="loadingDeleteAccount">
                    <v-card-title primary-title>Delete Account</v-card-title>
                    <v-card-text>
                        This action is <span class="error--text">irreversible</span>! Are you sure you want to delete
                        your account permanently?
                        <p class="caption error--text">{{deleteError}}</p>
                    </v-card-text>
                    <v-divider/>
                    <v-card-actions>
                        <v-spacer/>
                        <v-btn text @click="deleteDialog=false">Cancel</v-btn>
                        <v-btn color="error" text @click="deleteAccount">Delete Account</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </div>
        <h2>Other</h2>
        <p class="caption">To save data, network requests are saved to the cache and only refreshed periodically,
            instead of for each request.</p>
        <v-btn @click="clearCache()">Clear Cache</v-btn>
    </div>
</template>

<script>
    import Color from 'color-converter';
    import ApiKey from "../js/ApiKey";

    export default {
        name: 'Settings',
        components: {},
        data() {
            return {
                pwDialog: false,
                deleteDialog: false,
                password: '',
                color: '',
                pwError: '',
                deleteError: '',
                loadingChangePassword: false,
                loadingDeleteAccount: false,
                loadingLogout: false,
                slowCpu: this.$store.state.slowCpu,
                showVisual: localStorage.getItem('visualize') && localStorage.visualize === 'true',
                timeout: null,
                apiKey: '',
            }
        },
        mounted() {
            if (ApiKey.customKey)
                this.apiKey = ApiKey.key;
            this.color = this.$vuetify.theme.themes.dark.primary;
        },
        methods: {
            removeApiKey() {
                localStorage.removeItem('apiKey');
                this.apiKey = '';
            },
            setApiKey() {
                if (this.apiKey !== '') {
                    localStorage.apiKey = this.apiKey;
                    location.reload();
                } else {
                    alert("Fill in something first");
                }
            },
            async logout() {
                this.loadingLogout = true;
                await this.$store.dispatch('logout');
                this.loadingLogout = false;
            },
            async updatePassword(e) {
                e.preventDefault();
                this.loadingChangePassword = true;
                let result = await this.$store.dispatch('updatePassword', this.password);
                if (result !== true) {
                    this.pwError = result.message;
                } else {
                    this.pwDialog = false;
                    this.$emit('snack', 'Your password has been updated');
                }
                this.loadingChangePassword = false;
            },
            resetColor() {
                this.color = '#02c780';
            },
            async deleteAccount() {
                if (confirm("Are you sure you want to delete your account? This is irreversible!")) {
                    this.loadingDeleteAccount = true;
                    let result = await this.$store.dispatch('deleteAccount');
                    if (result !== true) {
                        this.deleteError = result.message;
                    } else {
                        this.deleteDialog = false;
                        this.$emit('snack', 'Your account has been deleted');
                    }
                    this.loadingDeleteAccount = false;
                }
            },
            clearCache() {
                localStorage.clear();
                location.reload();
            }
        },
        watch: {
            slowCpu() {
                this.$store.commit("setSlowCpu", this.slowCpu);
            },
            showVisual() {
                localStorage.visualize = this.showVisual;
            },
            color() {
                if (this.color) {
                    if (this.timeout !== null)
                        clearTimeout(this.timeout);
                    this.timeout = setTimeout(() => {
                        let color = Color.fromHex(this.color).lighten(7);
                        let newHue = (color.hue - 0.1);
                        if (newHue < 0)
                            newHue = newHue + 1;
                        if (newHue > 1)
                            newHue = newHue - 1;
                        let color2 = color.clone().setHue(newHue).lighten(7);

                        let colorBottom = color.toHex();
                        let colorTop = color2.toHex();
                        let canvas = this.$refs.canvas;
                        let img = this.$refs.favicon;
                        canvas.width = img.width;
                        canvas.height = img.height;
                        canvas.style.width = img.width + 'px';
                        canvas.style.height = img.height + 'px';
                        let context = canvas.getContext('2d');

                        context.drawImage(img, 0, 0);
                        context.globalCompositeOperation = 'source-in';

                        const grd = context.createLinearGradient(0, 0, 0, canvas.height);
                        grd.addColorStop(0, colorTop);
                        grd.addColorStop(1, colorBottom);

                        context.fillStyle = grd;
                        context.fillRect(0, 0, canvas.width, canvas.height);

                        let url = canvas.toDataURL('image/png');
                        let customColor = {
                            colorBottom: color.toHex(),
                            colorTop: color2.toHex(),
                            image: url,
                            width: canvas.width,
                            height: canvas.height,
                        };
                        this.$store.commit('setCustomColor', customColor);
                        localStorage.customColor = JSON.stringify(customColor);
                    }, 300);
                    this.$vuetify.theme.themes.dark.primary = this.color;
                    localStorage.color = this.color;
                }
            },
        },
    }
</script>

<style scoped>
    button {
        margin: 10px;
    }
</style>
