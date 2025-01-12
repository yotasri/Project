<template>
<v-container style="background-color: rgb(156, 255, 255);" fluid>
    <v-row>
        <!-- ส่วนควบคุมการทำงาน -->
        <v-col cols="12" md="3" style="padding-right: 50px;">
            <v-row>
                <v-col>
                    <div class="control-container">
                        <h1>Control Speaker</h1>
                        <!-- สวิตช์สำหรับควบคุมลำโพง (เปิด/ปิด) -->
                        <v-card-actions>
                            <v-switch v-model="isSpeakerOn" :label="isSpeakerOn ? 'Turn On' : 'Turn Off'" :color="isSpeakerOn ? 'blue' : 'blue'" @change="handleSpeakerToggle"></v-switch>
                        </v-card-actions>

                        <!-- สวิตช์สำหรับโหมดอัตโนมัติ (เปิด/ปิด) -->
                        <v-card-actions>
                            <v-switch style="margin-top: -50px;" v-model="isAutoMode" :label="isAutoMode ? 'Enable Auto Mode' : 'Disable Auto Mode'" :color="isAutoMode ? 'blue' : 'blue'" @change="handleAutoModeToggle"></v-switch>
                        </v-card-actions>

                        <h1> Status </h1><br>
                        <v-row>
                            <!-- แสดงสถานะของลำโพง (เปิด/ปิด) -->
                            <v-col cols="12">
                                <v-chip :color="isSpeakerOn ? 'green' : 'red'" label>
                                    {{ isSpeakerOn ? 'Speaker On' : 'Speaker Off' }}
                                </v-chip>
                            </v-col>
                            <!-- แสดงสถานะของโหมดอัตโนมัติ (เปิด/ปิด) -->
                            <v-col cols="12">
                                <v-chip :color="isAutoMode ? 'green' : 'red'" label>
                                    {{ isAutoMode ? 'Auto Mode On' : 'Auto Mode Off' }}
                                </v-chip>
                            </v-col>
                        </v-row>
                    </div>
                </v-col>
            </v-row>
        </v-col>

        <!-- ส่วนแสดงข้อมูลการตรวจจับ -->
        <v-col cols="12" md="9" style="margin-top: 50px;">
            <v-row>
                <v-col>
                    <v-card-title>
                        <h2 style="text-align: center;">Statistics</h2><br>
                    </v-card-title>
                    <v-card-text>
                        <v-row>
                            <!-- แสดงจำนวนการตรวจจับทั้งหมด -->
                            <v-col>
                                <v-card class="pa-4" color="blue-grey lighten-2">
                                    <v-card-title>Total Detections</v-card-title>
                                    <v-card-text class="display-1">{{ totalDetections }}</v-card-text>
                                </v-card>
                            </v-col>
                            <!-- แสดงการตรวจจับล่าสุด -->
                            <v-col>
                                <v-card class="pa-4" color="green lighten-2">
                                    <v-card-title>Last Detected</v-card-title>
                                    <v-card-text>{{ lastDetected }}</v-card-text>
                                </v-card>
                            </v-col>
                        </v-row>
                    </v-card-text>
                </v-col>
            </v-row>

            <!-- แสดงข้อมูลการตรวจจับในตาราง -->
            <v-row>
                <v-col>
                    <v-card-title>
                        <h2 style="text-align: center;">Detection Data</h2><br>
                    </v-card-title>
                    <v-data-table :headers="headers" :items="detections" class="elevation-1" :items-per-page="5" :pagination.sync="pagination">
                        <template v-slot:no-data>
                            <v-alert type="info" :value="true">
                                No data available
                            </v-alert>
                        </template>
                    </v-data-table>
                </v-col>
            </v-row>
        </v-col>
    </v-row>
</v-container>
</template>

<script>
import mqtt from 'mqtt';

definePageMeta({
    layout: "iot",
});

export default {
    data() {
        return {
            detections: [], // ข้อมูลการตรวจจับที่ได้รับจาก MQTT
            headers: [{
                    text: 'Order',
                    value: 'id'
                },
                {
                    text: 'Date and Time',
                    value: 'detected_time'
                },
                {
                    text: 'Detected',
                    value: 'detected'
                },
            ],
            isSpeakerOn: false, // สถานะของลำโพง
            isAutoMode: false, // สถานะของโหมดอัตโนมัติ
            client: null, // ตัวแปรเชื่อมต่อกับ MQTT broker
            pagination: {
                sortDesc: true,
            },
            totalDetections: 0, // จำนวนการตรวจจับทั้งหมด
            lastDetected: 'N/A', // เวลาและวันที่การตรวจจับล่าสุด
        };
    },
    created() {
        this.connectToMQTT(); // เชื่อมต่อกับ MQTT broker เมื่อคอมโพเนนต์ถูกสร้าง
    },
    methods: {
        // ฟังก์ชันเชื่อมต่อกับ MQTT broker
        connectToMQTT() {
            this.client = mqtt.connect('ws://broker.hivemq.com:8000/mqtt'); // หรือ URL ของ MQTT Broker ที่คุณใช้งาน

            // ตรวจสอบเมื่อเชื่อมต่อสำเร็จ
            this.client.on('connect', () => {
                console.log('Connected to MQTT broker');
                // หากเชื่อมต่อสำเร็จ, ให้สมัครหัวข้อที่ต้องการ
                this.client.subscribe('speaker/manual');
                console.log('Subscribe to speaker/manual Success');
            });

            // ตรวจสอบกรณีเกิดข้อผิดพลาดในการเชื่อมต่อ
            this.client.on('error', (error) => {
                console.error('Failed to connect to MQTT broker:', error);
            });
        },

        //ฟังก์ชันเพิ่มข้อมูลการตรวจจับใหม่
        addDetection(data) {
            const newDetection = {
                id: this.detections.length + 1,
                detected_time: new Date().toLocaleString(), // เวลาที่ตรวจจับ
                detected: data.detected, // สถานะการตรวจจับ
            };
            this.detections.push(newDetection); // เพิ่มข้อมูลใหม่เข้าสู่ตาราง
            this.updateStatistics(newDetection); // อัปเดตสถิติการตรวจจับ
        },

        // ฟังก์ชันควบคุมการเปิด/ปิดลำโพง
        handleSpeakerToggle() {
            const command = this.isSpeakerOn ? '1' : '0'; // สถานะเปิด/ปิด
            this.client.publish('speaker/manual', command, (err) => {
                if (err) {
                    console.error('Error sending message:', err);
                } else {
                    console.log(`Manaul mode status: ${command}`);
                }
            });
        },

        // ฟังก์ชันควบคุมการเปิด/ปิดโหมดอัตโนมัติ
        handleAutoModeToggle() {
            if (this.isSpeakerOn) {
                // ถ้าลำโพงเปิดอยู่ จะไม่สามารถเปิดโหมดอัตโนมัติได้
                this.isAutoMode = false;
                console.log('Auto mode disabled because speaker is on');
                // สามารถแสดงข้อความแจ้งเตือนหรือไม่ให้ผู้ใช้เปิดโหมดอัตโนมัติ
                alert('Cannot enable Auto Mode while speaker is ON');
            } else {
                // ถ้าลำโพงปิดอยู่ สามารถเปิดโหมดอัตโนมัติได้
                if (this.isAutoMode) {
                    // เมื่อเปิดโหมดอัตโนมัติ, ส่งค่า '3' ไปยัง MQTT
                    this.client.publish('speaker/manual', '3', (err) => {
                        if (err) {
                            console.error('Error sending auto mode ON message:', err);
                        } else {
                            console.log('Auto mode enabled');
                        }
                    });
                } else {
                    // เมื่อปิดโหมดอัตโนมัติ, ส่งค่า '4' ไปยัง MQTT
                    this.client.publish('speaker/manual', '4', (err) => {
                        if (err) {
                            console.error('Error sending auto mode OFF message:', err);
                        } else {
                            console.log('Auto mode disabled');
                        }
                    });
                }
            }
        },

        // ฟังก์ชันอัปเดตสถิติการตรวจจับ
        updateStatistics(newDetection) {
            this.totalDetections += 1;
            this.lastDetected = newDetection.detected_time;
        },
    },
};
</script>

<style scoped>
.control-container {
    background-color: #ffffff;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    border: 2px solid #797979;
    height: 710px;
    overflow: auto;
}
</style>
