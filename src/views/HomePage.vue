<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Blank</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">BLE Sample App</ion-title>
        </ion-toolbar>
      </ion-header>

      <div id="container">
        <div>
          Location Enabled:
          <span>{{ getLocationEnabledStatus() }}</span>
        </div>
        <div>
          BLE Enabled:
          <span>{{ getBleEnabledStatus() }}</span>
        </div>
        <div>
          Is Connected:
          <span>{{ getConnectedStatus() }}</span>
        </div>
        <div>
          Is Connecting:
          <span>{{ getIsConnectingStatus() }}</span>
        </div>
        <div>
          Is Disconnecting:
          <span>{{ getIsDisconnectingStatus() }}</span>
        </div>
        <ul style="border: 1px solid black; max-height: 300px; overflow-y: scroll;">
          <li v-for="device in devices"
              :key="device.id"
              :style="getDeviceStyle(device)"
              @click="interactWithDevice(device)">
            {{ device.id }} - {{ device.name }}
          </li>
        </ul>
        <ion-button v-if="isScanning !== true" @click="startScan">
          SCAN
        </ion-button>
        <ion-button v-if="isScanning === true" @click="stopScan">
          STOP SCAN
        </ion-button>
        <ion-button v-if="connectedDevice !== null" @click="write('DEVICE_INFO')">
          SEND WRITE COMMAND
        </ion-button>
        <div>
          Last event:
          <span> {{ lastEvent }}</span>
        </div>
        <div>
          Command results:
          <span> {{ commandResults }}</span>
        </div>
        <ion-button @click="updateStatus">
          CHECK LOCATION & BLE ENABLED
        </ion-button>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonButton,

} from '@ionic/vue';
import { defineComponent } from 'vue';

declare const window: any;

const SERVICE_ID = 'D3DADCBA-E4FA-4016-BA33-8BCC671999A7';
const RN_CHAR_ID = '1A42';
const W_CHAR_ID = '1A44'

export default defineComponent({
  name: 'HomePage',
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonButton,
    IonToolbar
  },

  data() {
    let isScanning: any = null,
        isLocationEnabled: any = null,
        isBleEnabled: any = null,
        isConnecting: any = null,
        isDisconnecting: any = null,
        lastEvent: any = null,
        commandResults: any = null,
        connectedDevice: any = null,
        requestNumber: number = 0,
        devices: any[] = [];

    return {
      isScanning: isScanning,
      //start null since we dont know
      isBleEnabled: isBleEnabled,
      isLocationEnabled: isLocationEnabled,
      isConnecting: isConnecting,
      isDisconnecting: isDisconnecting,
      //ble devices
      devices: devices,
      lastEvent: lastEvent,
      connectedDevice: connectedDevice,
      requestNumber: requestNumber,
      commandResults: commandResults
    }
  },

  created() {
    this.updateStatus();
  },

  methods: {
    getDeviceStyle(device) {
      let styles = [
        'padding: 10px 0;',
        'border: 1px solid grey;'
      ];

      if (device === this.connectedDevice) {
        styles.push('background: green; color: white;')
      }

      return styles;
    },

    updateStatus() {

      this.checkIsBleEnabled();
      this.checkIsLocationEnabled();
    },

    getLocationEnabledStatus() {
      switch (this.isLocationEnabled) {
        case null:
          return 'Unknown';
        case true:
          return 'Yes';
        case false:
          return 'No';
      }
    },

    getBleEnabledStatus() {
      switch (this.isBleEnabled) {
        case null:
          return 'Unknown';
        case true:
          return 'Yes';
        case false:
          return 'No';
      }
    },

    getConnectedStatus() {
      switch (this.connectedDevice) {
        case null:
          return 'Unknown';
      }

      return this.connectedDevice.id + ' - ' + this.connectedDevice.name;
    },

    getIsConnectingStatus() {
      switch (this.isConnecting) {
        case null:
          return 'Unknown';
        case true:
          return 'Yes';
        case false:
          return 'No';
      }
    },

    getIsDisconnectingStatus() {
      switch (this.isDisconnecting) {
        case null:
          return 'Unknown';
        case true:
          return 'Yes';
        case false:
          return 'No';
      }
    },

    connectToDevice(device) {
      this.lastEvent = 'Connecting to BLE device';
      this.isConnecting = true;
      window.ble.connect(device.id,
          (device) => {
            console.log('ble.connect:success:', device);
            this.lastEvent = 'Connecting to BLE device success';
            this.isConnecting = false;
            this.connectedDevice = device;
            this.startNotification(device);
          },
          (error) => {
            this.isConnecting = false;
            this.lastEvent = 'Connecting to BLE device error';
            console.log('ble.connect:error:', error);
          }
      );
    },

    startNotification(device) {
      this.lastEvent = 'Registering startNotification';

      window.ble.startNotification(device.id,
          SERVICE_ID,
          RN_CHAR_ID,
          (success) => {
            console.log('ble.startNotification:success:', success);

            let response = String.fromCharCode(...new Uint8Array(success));
            this.commandResults = response;
            this.lastEvent = 'Registering startNotification success';
          },
          (error) => {
            this.lastEvent = 'Registering startNotification error';
            console.log('ble.startNotification:error:', error);
          }
      );
    },

    write(command) {
      this.lastEvent = 'writing BLE command';
      let device = this.connectedDevice,
          /**
           * Tweak this message as you need, for our BLE device it expects this pattern
           */
          message = String.fromCharCode(this.requestNumber) + command + '\0',
          bytes = new ArrayBuffer(message.length * 2),
          bytesUint16 = new Uint16Array(bytes);

      //increment request
      this.requestNumber++;

      for (var i = 0; i < message.length; i++) {
        bytesUint16[i] = message.charCodeAt(i);
      }

      let newBytes = new Uint8Array(bytesUint16),
          encodedString = btoa(String.fromCharCode.apply(null,
              [...new Uint8Array(newBytes)]
              // newBytes
          ));

      window.ble.write(device.id, SERVICE_ID, W_CHAR_ID, encodedString,
          (success) => {
            console.log('ble.write:success:', success);
            this.lastEvent = 'writing BLE command success';
          },
          (error) => {
            console.log('ble.write:error:', error);
            this.lastEvent = 'writing BLE command error';
          }
      );
    },

    checkIsBleEnabled() {
      this.lastEvent = 'Checking BLE enabled';
      window.ble.isEnabled(
          (success) => {
            console.log('ble.isEnabled:success:', success);
            this.isBleEnabled = true;
            this.lastEvent = 'BLE enabled success';
          },
          (error) => {
            console.log('ble.isEnabled:error:', error);
            this.isBleEnabled = false;
            this.lastEvent = 'BLE enabled error';
          }
      );
    },

    checkIsLocationEnabled() {
      this.lastEvent = 'Checking Location enabled';
      window.ble.isLocationEnabled(
          (success) => {
            console.log('ble.isLocationEnabled:success:', success);
            this.lastEvent = 'Checking Location enabled success';
            this.isLocationEnabled = true;
          },
          (error) => {
            console.log('ble.isLocationEnabled:error:', error);
            this.isLocationEnabled = false;
            this.lastEvent = 'Checking Location enabled error';
          }
      );
    },

    interactWithDevice(device) {
      console.log('interactWithDevice:', device);

      if (device === this.connectedDevice) {
        //disconnect
      } else {
        //connect
        this.stopScan();
        this.connectToDevice(device);
      }
    },

    startScan() {
      this.lastEvent = 'Start scanning BLE';
      //reset devices
      this.devices = [];

      window.ble.startScan([],
          (device) => {
            this.isScanning = true;
            console.log('ble.startScan:success:', device);
            this.devices.push(device);
            this.lastEvent = 'Start scanning BLE success';
          },
          (error) => {
            console.log('ble.startScan:error:', error);
            this.lastEvent = 'Start scanning BLE error';
          }
      );
    },

    stopScan() {
      this.lastEvent = 'Stop scanning BLE';
      window.ble.stopScan(
          (success) => {
            this.isScanning = false;
            console.log('ble.stopScan:success:', success);
            this.lastEvent = 'Stop scanning BLE success';
          },
          (error) => {
            console.log('ble.stopScan:error:', error);
            this.lastEvent = 'Stop scanning BLE error';
          }
      );
    }
  }
});
</script>

<style scoped>
#container {
  text-align: center;

  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;

  color: #8C8C8C;

  margin: 0;
}

#container a {
  text-decoration: none;
}
</style>
