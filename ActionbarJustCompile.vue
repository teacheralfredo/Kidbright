<template>
    <div>
        <v-tooltip bottom>
            <v-btn color="primary darken-2" slot="activator" icon @click="compileDialog = true">
                <v-icon dark>fa-check</v-icon>
            </v-btn>
            <span>Just Compile</span>
        </v-tooltip>

        <v-dialog v-model="compileDialog" persistent max-width="450px">
            <v-card>
                <v-card-title>
                    <span class="headline">Compile &#x26; Run</span>
                </v-card-title>

                <v-card-text>
                    <v-container>
                        <v-layout align-center column>
                            <v-flex xs12>
                                <v-progress-circular v-if="compileStep < 3"
                                                     :size="80"
                                                     :width="8"
                                                     color="primary"
                                                     indeterminate>
                                </v-progress-circular>
                                <v-fade-transition :hide-on-leave="true">
                                    <v-icon color="green" size="110" v-if="compileStep >= 3">
                                        check_circle_outline
                                    </v-icon>
                                </v-fade-transition>
                            </v-flex>
                        </v-layout>
                    </v-container>
                    <v-flex xs12>
                        <v-stepper v-model="compileStep" vertical class="elevation-0 pb-0">
                            <v-stepper-step step="1" :complete="compileStep > 1"
                                            :rules="[()=>{ return stepResult['1'].result }]">
                                Find KidBright
                                <small v-if="compileStep > 1">{{stepResult["1"].msg}}</small>
                            </v-stepper-step>
                            <v-stepper-content step="1" v-if="compileStep >= 1">
                                {{stepResult["1"].msg}}
                            </v-stepper-content>

                            <v-stepper-step step="2" :complete="compileStep > 2"
                                            :rules="[()=>{ return stepResult['2'].result }]">
                                Compile the code
                                <small v-if="compileStep > 2">{{stepResult["2"].msg}}</small>
                            </v-stepper-step>
                            <v-stepper-content step="2" v-if="compileStep >= 2">
                                {{stepResult["2"].msg}}
                            </v-stepper-content>
                        </v-stepper>
                    </v-flex>
                </v-card-text>
                <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn color="blue darken-1" flat @click="rebuild"
                           :disabled="compileStep < 2 && failed === false">Compile
                    </v-btn>
                    <v-btn color="blue darken-1" flat @click="compileDialog = false"
                           :disabled="compileStep < 2 && failed === false">Close
                    </v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </div>
</template>

<script>
  const engine = Vue.prototype.$engine;
  const G = Vue.prototype.$global;
  var path = `${engine.util.boardDir}/${G.board.board}/compiler`;
  var boardCompiler = engine.util.requireFunc(path);

  var comport = "";
  var mac = "";
  var boardName = "";
  var mother = null;
  export default {
    data() {
      return {
        compileStep: 1,
        compileDialog: false,
        failed: false,
        stepResult: {
          "1": {
            result: true,
            msg: "",
          },
          "2": {
            result: true,
            msg: "",
          },
          "3": {
            result: true,
            msg: "",
          },
        },
      };
    },

    created() {
      mother = this;
    },

    beforeDestroy() {

    },
    methods: {
      rebuild() {
        mother.compileStep = 1;
        mother.failed = false;
        mother.stepResult["1"].result = true;
        mother.stepResult["2"].result = true;
        mother.stepResult["3"].result = true;
        Vue.nextTick(mother.run);
      },
      run() { //find port and mac
        console.log("---> step 1 <---");
        G.$emit("compile-begin");
        mother.compileStep = 1;
        mother.stepResult["1"].msg = "Compiling..";
        //this.stepResult["1"].msg += ` at ${comport}`;
        const p = new Promise((resolve, rejecf) => {
          resolve({mac: "ff:ff:ff:ff:ff:ff"});
        });
        p.then(boardMac => {
          //this.stepResult["1"].msg += ` MAC ${boardMac.mac}`;
          mac = boardMac.mac;
          boardName = mac.replace(/:/g, "-");
          console.log(`[STEP 1] got it boardName = ${boardName} mac = ${mac}`);
          mother.compileStep = 2;
          console.log("---> step 2 test <---");

          this.stepResult["2"].msg = "Compile board ... ";
          var rawCode = G.editor.rawCode;
          var config = {
            board_mac_addr: mac,
            sta_ssid: this.$global.board.package["kidbright-actionbar"].wifi_ssid,
            sta_password: this.$global.board.package["kidbright-actionbar"].wifi_password,
            enable_iot: this.$global.board.package["kidbright-actionbar"].enable_iot,
          };

          let compileCb = (status) => {
            mother.stepResult["2"].msg = status;
          };
          return boardCompiler.compile(rawCode, boardName, config, compileCb);
        }).then(() => {
          mother.stepResult["2"].msg += "done!";
          mother.compileStep = 3;
          G.$emit("compile-success");
        }).catch(err => {
          console.log("------ process error ------");
          engine.util.compiler.parseError(err).then(errors => {
            console.error(`errors:`, errors);
            G.$emit("compile-error",errors);
            if (mother.compileStep == 1) {
              mother.stepResult["1"].msg = "Cannot find KidBright : " + err;
              mother.stepResult["1"].result = false;
            } else if (mother.compileStep == 2) {
              mother.stepResult["2"].msg = `${errors.join("\n")}`;
              mother.stepResult["2"].result = false;
            } else if (mother.compileStep == 3) {
              mother.stepResult["3"].msg = "Cannot upload program : " + err;
              mother.stepResult["3"].result = false;
            }
            mother.failed = true;
          }).catch(errors => {
            console.log("errors", errors);
            //G.$emit("compiler-error", errors);
            mother.failed = true;
          });
          mother.failed = true;
        });
      },
    },
    watch: {
      "compileDialog": function(val) {
        if (val) {//on opening
          this.rebuild();
        }
      },
    },
  };
</script>
