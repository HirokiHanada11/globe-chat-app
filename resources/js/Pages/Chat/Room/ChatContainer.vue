<template>
    <chat-three-container :messages="messages" :room="currentRoom" :activeUsers="activeUsers" :cameraNum="camera" >
        <template #header>
            <img :src="currentRoom.photo" class="h-12 w-12 mx-4 float-left rounded-full"/>
            <h1 class="font-semibold text-4xl leading-tight p-3 truncate">
                <span @click="showModal = true" class="cursor-pointer hover:underline text-gray-200 "><b>{{currentRoom.name}}</b> --{{currentRoom.description}}</span>
            </h1>
        </template>

        <template #footer>
            <div v-if="alert.user" class="w-full justify-center flex"
                v-bind:class="{'animate-fade-in-up': alert.show, 'animate-fade-out-down': !alert.show}">
                <span><b>{{alert.user.name}}</b>{{alert.message}}</span>
            </div>                
            <input-message 
                :room="currentRoom" 
                v-on:sending="appendNewMessage"
                v-on:messagefailed="messageFailedToSend"
                v-on:demostarted="demoStart"
                v-on:demostopped="demoStop"
                />
        </template>

        <template #activeUsersContainer>
            <active-users-container 
                v-show="showActive.state != 'hide'" 
                :activeUsers="activeUsers"
                v-bind:class="{
                    'animate-fade-in-left': showActive.state == 'show',
                    'animate-fade-out-right': showActive.state == 'hiding', 
                }"/>
            <div class="h-full w-1/12 absolute top-0 left-0 bg-gray-300 opacity-5 hover:opacity-50" @click="toggleActive"></div>
        </template>

        <template #messagesContainer>
            <message-container 
                v-show="showMessages.state != 'hide'" 
                :messages="messages" 
                :fetching="fetching" 
                v-on:resend="resendMessage" 
                v-on:fetchmoremessages="getPaginatedMessages"
                v-bind:class="{
                    'animate-fade-in-right': showMessages.state == 'show',
                    'animate-fade-out-left': showMessages.state == 'hiding', 
                }" />
        <div class="h-full w-1/12 absolute top-0 right-0 bg-gray-300 opacity-5 hover:opacity-50" @click="toggleMessage"></div>
        </template>
    </chat-three-container>
        <!-- <div class="bg-transparent absolute top-0 w-full flex justify-center ">
            <input 
                v-if="!camera"
                type="text"
                v-model="topic"
                @keyup.enter="fetchNews(topic)"
                placeholder="Search News by keyword"
                class="bg-blue-900 border border-transparent focus:outline-none focus:border-none rounded-full text-white"
            />
            <button
                v-if="!camera"
                @click="fetchNews(currentRoom.topic)"
                class="bg-blue-900 border border-transparent hover:outline-2 hover:border-none rounded-full text-white p-2">
                Fetch News On Room Topic
            </button>
            <button
                v-if="camera"
                @click="setBackCamera()"
                class="bg-blue-900 border border-transparent hover:outline-2 hover:border-none rounded-full text-white p-2">
                Fetch News
            </button>
        </div>  -->
        <!-- <div class="bg-transparent absolute bottom-0 w-full flex justify-center ">
            <button
                v-if="!camera"
                @click="setBackCamera()"
                class="bg-blue-900 border border-transparent hover:outline-2 hover:border-none rounded-full text-white p-2">
                Back To Chat
            </button>
        </div>  -->

    <room-settings-modal :showModal="showModal" :currentRoom="currentRoom" :subscribers="subscribers" :activeUsers="activeUsers" v-on:close="showModal = false" />

</template>

<script>
    import AppLayout from '@/Layouts/AppLayout'
    import MessageContainer from './MessageContainer.vue'
    import InputMessage from './InputMessage.vue'
    import ChatThreeContainer from './ChatThreeContainer.vue'
    import ActiveUsersContainer from './ActiveUsersContainer.vue'
    // import NewsContainer from './NewsContainer.vue'
    // import { prefecToCoords } from '../ThreeJS/japanPrefecture'
    import RoomSettingsModal from './RoomSettingsModal.vue'

    export default {
        props: ['roomId'],
        components: {
            AppLayout,
            MessageContainer,
            InputMessage,
            ChatThreeContainer,
            ActiveUsersContainer,
            // NewsContainer,
            RoomSettingsModal,
        },
        data: () => {
            return {
                currentRoom: new Object(),
                messages: new Array(),
                activeUsers: {
                    changeType: 'default',
                    subjectUser: null,
                    usersList: new Array(),
                },
                subscribers: new Array(),
                showActive: {
                    state: 'show',
                    timeOut: undefined,
                },
                showMessages: {
                    state: 'show',
                    timeOut: undefined,
                },
                // showNews: {
                //     state: 'hide',
                //     timeOut: undefined,
                // },
                showModal: false,
                chatScrollPosition: 0,
                sortBy: 'popularity',
                // news: new Array(),
                // topic: '',
                camera: true,
                demoInterval: 0,
                pagination: 0,
                newestMessageId: 0,
                fetching: false,
                alert: {
                    show: false,
                    user: null,
                    message: null, 
                    timestamp: null,
                },
            }
        },
        async beforeUnmount() {
            await this.disconnectHandler();
            document.removeEventListener('backbutton', this.disconnectHandler);
        },
        async mounted() {
            document.addEventListener('backbutton', this.disconnectHandler);
            await this.activateUser();
            this.currentRoom = await this.getCurrentRoom();
            this.activeUsers = await {
                changeType: 'default',
                subjectUser: null,
                usersList: this.currentRoom.active_users,
            };
            this.subscribers = await this.currentRoom.users;
            // console.log(this.activeUsers);
            await this.connect();
        },
        methods: {
            //room method
            async getCurrentRoom(){ //fetch the information about the current room
                try { 
                    let response = await axios.get(`/chat/currentroom/${this.roomId}`);
                    return response.data;                    
                } catch (error) {
                    console.error(error);
                }
            },

            //new active user handler
            async newUserHandler(user){
                let data = await this.getCurrentRoom();
                this.activeUsers = await {
                    changeType: 'create',
                    subjectUser: data.active_users.find(activeUser => activeUser.id === user.id),
                    usersList: data.active_users,
                };
                this.subscribers = await data.users;
            },

            //leaving user handler
            removeUserHandler(user){
                let subjectUser;
                let newList = new Array();
                for(let i = 0; i < this.activeUsers.usersList.length; i++){
                    if (this.activeUsers.usersList[i].id === user.id){
                        subjectUser = this.activeUsers.usersList[i];
                        newList = this.activeUsers.usersList.slice(0, i).concat(this.activeUsers.usersList.slice(i + 1))
                        break;
                    }
                }

                this.activeUsers = {
                    changeType: 'remove',
                    subjectUser: subjectUser,
                    usersList: newList,
                };
            },

            //alert for user joining and leaving
            createAlert(user, type){
                if(this.alert.timeOut){
                    clearTimeout(this.alert.timeOut);
                }
                this.alert = {
                    show: true,
                    user: user,
                    message: type ? ' joined the chat!' : ' left the chat', 
                    timestamp: Date.now(),
                    timeOut: setTimeout(() => {
                        this.alert.show = false;
                        this.alert.timeOut = undefined;
                        setTimeout(() => this.alert.user = null, 500);
                    }, 3000),
                }
                
            },
            
            //Broadcasting methods
            connect(){ //connects the user to the room channel
                if( this.currentRoom.id ){
                    let vm = this;
                    console.log('connecting')
                    window.Echo.join(`chat.${this.currentRoom.id}`)
                    .here((users)=>{
                        this.getPaginatedMessages();
                    })
                    .joining((user) => {
                        this.createAlert(user, 1);
                        this.newUserHandler(user);
                    })
                    .leaving((user) => {
                        this.createAlert(user, 0);
                        this.removeUserHandler(user);
                    })
                    .listen('NewChatMessage', e => {
                        vm.getNewestMessage();
                    });
                }
            },
            disconnect( room ){//disconnect the user from the room channel 
                window.Echo.leave(`chat.${room.id}`);
            },
            
            //activation methods
            async activateUser(){
                try{
                    let response = await axios.post(`/activate/${this.roomId}/${this.$page.props.user.id}`)
                    if(response.status == 201){
                        console.log('activated user');
                    }
                }catch(error){
                    console.error(error);
                }
            },
            async deactivateUser(){
                try {
                    let response = await axios.post(`/deactivate/${this.currentRoom.id}/${this.$page.props.user.id}`);
                    if(response.status == 201){
                        console.log('deactivated user');
                    }
                } catch (error) {
                    console.error(error);
                }
            },

            async disconnectHandler (){                
                await this.deactivateUser();
                await this.disconnect(this.currentRoom);
            },

            //toggling methods for view
            setBackCamera() { //initiates the camera movement
                this.camera = !this.camera;
            },

            toggleMessage() { //show and hide message column
                if(this.showMessages.state === 'show'){
                    this.showMessages.state = 'hiding';
                    this.showMessages.timeOut = setTimeout(() => this.showMessages.state = 'hide', 300);
                }
                else if(this.showMessages.state === 'hiding'){
                    this.showMessages.state = 'show'
                    clearTimeout(this.showMessages.timeOut);
                    this.showMessages.timeOut = undefined;
                }
                else if(this.showMessages.state === 'hide'){
                    this.showMessages.state = 'show'
                }
            },

            toggleActive() { //show and hide active users column
                if(this.showActive.state === 'show'){
                    this.showActive.state = 'hiding';
                    this.showActive.timeOut = setTimeout(() => this.showActive.state = 'hide', 300);
                }
                else if(this.showActive.state === 'hiding'){
                    this.showActive.state = 'show'
                    clearTimeout(this.showActive.timeOut);
                    this.showActive.timeOut = undefined;
                }
                else if(this.showActive.state === 'hide'){
                    this.showActive.state = 'show'
                }
            },

            //message methods
            async getPaginatedMessages(){//called when user reachs max scroll       
                this.fetching = true;
                try {
                    let response = await axios.get(`/chat/room/${this.currentRoom.id}/paginated/${this.pagination}`);
                        if(response.data.length > 0){
                            this.messages = await response.data.reverse().concat(this.messages);
                            if (this.pagination == 0){
                                this.newestMessageId = await this.messages[this.messages.length-1].id;
                            }
                            this.pagination += 1;
                        }
                            this.fetching = false; 
                } catch (error) {
                   console.error(error);
                    this.fetching = false;   
                }   
            },

            async getNewestMessage(){//gets the latest message when new message event occurs
                try{
                let response = await axios.get(`/chat/room/${this.currentRoom.id}/newestmessage`);
                    let newMessage = await [response.data];
                    this.messages = await this.messages.concat(newMessage);
                    this.newestMessageId = await this.messages[this.messages.length-1].id;
                } catch(error) {
                    console.error(error);
                }
            },

            appendNewMessage(value){ //appends new message to the message data 
                let newMessage = [{
                    id: this.newestMessageId + 1,
                    user: this.$page.props.user,
                    user_id: this.$page.props.user.id,
                    chat_room_id: this.currentRoom.id,
                    message: value.message,
                    link: value.link,
                    article: value.article,
                    replying_to: value.replyTo,
                    created_at: Date.now(),
                }];
                this.messages = this.messages.concat(newMessage);
                this.newestMessageId = this.messages[this.messages.length-1].id;
            },

            messageFailedToSend(){//error display
                let erroredMessage = this.messages.find(message => message.user_id === this.$page.props.user.id);
                erroredMessage.error = true;
            },

            resendMessage(value){//ressend message
                let payload = {
                    message: value.message,
                    link: value.link,
                    article: value.article,
                    replyTo: value.replyTo,
                }
                
                axios.post(`/chat/room/${this.room.id}/message`, payload)
                .then( response => {
                    if( response.status == 201 ){
                        console.log('message sent');
                        this.messages.find(message => message.id === value.id).error = undefined;
                    }
                })
                .catch( error => {
                    console.error(error);
                })
            },

            //news search methods
            // fetchNews(topic) {
            //     this.showNews.state = 'show';
            //     if(topic != ''){
            //         axios.get(`/chat/room/news/${encodeURI(topic)}`)
            //         .then( response => {
            //             console.log(response.data.articles);
            //             this.news = response.data.articles;
            //             this.topic = '';
            //         })
            //         .catch( error => {
            //             console.error(error);
            //         })
            //     }
            // },
            // searchNews(topic) {
            //     this.showNews.state = 'show';
            //     if(topic != ''){
            //         axios.get(`/chat/room/news/search/${encodeURI(topic)}`)
            //         .then( response => {
            //             console.log(response.data.articles);
            //             this.news = response.data.articles;
            //             this.topic = '';
            //         })
            //         .catch( error => {
            //             console.error(error);
            //         })
            //     }
            // },
            demoStart(){
                if(this.currentRoom.owner.id === this.$page.props.user.id)
                {
                    this.demoInterval = setInterval(()=>{
                        if(Math.random() > 0.7){
                            let randomIndex = Math.floor(Math.random() * (this.activeUsers.usersList.length - 1) + 1);
                            let newMessage = [{
                                id: this.newestMessageId + 1,
                                user: this.activeUsers.usersList[randomIndex],
                                user_id: this.activeUsers.usersList[randomIndex].id,
                                chat_room_id: this.currentRoom.id,
                                message: '8'.repeat(Math.floor(Math.random()* 8 + 4)),
                                link: false,
                                article: null,
                                replying_to: null,
                                created_at: Date.now(),
                            }];
                            console.log(this.activeUsers.usersList, newMessage);
                            this.messages = this.messages.concat(newMessage);
                            this.newestMessageId = this.messages[this.messages.length-1].id;
                        }
                    },200)
                }
            },
            demoStop(){
                clearInterval(this.demoInterval);
                this.demoInterval = 0;
            }
            

            //Demo Methods
            // async startDemo(){
            //     const randomComment = ['私は南極に行く','軽く死ねますね','アメンボ赤いな愛ゆえに','選択肢はずっとあったよでも選んだんだよ、ここを選んだんだよ自分で','うるかにしてください','プリンは飲み物'];
            //     for(let i = 2; i < 6; i++){
            //         try {
            //             let demoRegion = Object.keys(prefecToCoords)[Math.floor(Math.random() * 46)]
            //             let response = await axios.post(`/activate/${this.currentRoom.id}/${i}`, {
            //                 region: demoRegion,
            //             });
            //             if( response.status == 200 ){
            //                 console.log('activated new demo user', response.data);
            //             }
            //         } catch (error) {
            //             console.error(error);
            //         }
            //         await this.getCurrentRoom();
            //     }
            //     this.demoInterval = await setInterval(()=>{
            //         let id = Math.floor(Math.random() * (6 - 2) + 2);
            //         let randomIndex = Math.floor(Math.random()* (randomComment.length - 1));
            //         let comment = randomComment[randomIndex];
            //         console.log(comment)
            //         axios.post(`/chat/room/${this.currentRoom.id}/demomessage`, {
            //             demoUserId: id,
            //             message: comment,
            //             link: false,
            //             article: null,
            //             replyTo: null,
            //         })
            //         .then( response => {
            //             if( response.status == 201 ){
            //                 console.log('Dummy message', response.data);
            //                 this.getNewestMessage();
            //             }
            //         })
            //         .catch( error => {
            //             console.error(error);
            //         })
            //     },3000);
            // },
            // stopDemo(){
            //     clearInterval(this.demoInterval);
            //     this.demoInterval = 0;
            //     for(let i = 2; i < 6; i++){
            //         axios.post(`/deactivate/${this.currentRoom.id}/${i}`)
            //         .then( response => {
            //             if( response.status == 201 ){
            //                 console.log('deactivated demo users');
            //             }
            //         })
            //         .catch( error => {
            //             console.error(error);
            //         })
            //     }
            //     this.getCurrentRoom();
            // },
            
        },
        
    }
</script>
