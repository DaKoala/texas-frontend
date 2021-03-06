<template>
    <div class="room">
        <div class="left">
            <div class="room__name">Room: {{ room.name }}</div>
            <div class="announcement">
                <div>{{ announcement }}</div>
                <div>
                    <font-awesome-icon :icon="['fas', 'funnel-dollar']"></font-awesome-icon>
                    <span>{{ pot }}</span>
                </div>
            </div>
            <div class="public">
                <PokerCard v-for="card in publicCards"
                           :card="card"
                           :key="String(card.point) + String(card.suit)"/>
            </div>
            <div class="player--me"
                 :class="{'player--fold': me.isFold,
                 'player--inturn': me.username === playerInTurn}">
                <div>
                    <font-awesome-icon :icon="['fas', 'user']"></font-awesome-icon>
                    {{ me.username }}
                </div>
                <div>
                    <font-awesome-icon :icon="['fas', 'coins']"></font-awesome-icon>
                    {{ me.chip }}
                </div>
                <div>
                    <font-awesome-icon :icon="['fas', 'hand-holding-usd']"></font-awesome-icon>
                    {{ me.betChip }}
                </div>
            </div>
            <div class="panel">
                <div class="panel__buttons">
                    <div class="panel__button"
                         v-if="options.fold"
                         @click="clickFold">Fold</div>
                    <div class="panel__button panel__button--green"
                         v-if="options.raise"
                         @click="clickRaise">Raise</div>
                    <div class="panel__button panel__button--green"
                         v-if="options.call"
                         @click="clickCall">Call</div>
                    <div class="panel__button panel__button--green"
                         v-if="options.check"
                         @click="clickCheck">Check</div>
                </div>
                <div class="panel__cards">
                    <PokerCard v-for="card in me.cards"
                               :key="String(card.point) + String(card.suit)"
                               :card="card"></PokerCard>
                </div>
            </div>
            <div v-for="(player, index) in otherPlayers"
                 :key="player.username + player.chip"
                 class="player"
                 :class="{'player--left': index === 0,
                 'player--fold': player.isFold,
                 'player--right': index === 1,
                 'player--inturn': player.username === playerInTurn}">
                <div>
                    <font-awesome-icon :icon="['fas', 'user']"></font-awesome-icon>
                    {{ player.username }}
                </div>
                <div>
                    <font-awesome-icon :icon="['fas', 'coins']"></font-awesome-icon>
                    {{ player.chip }}
                </div>
                <div>
                    <font-awesome-icon :icon="['fas', 'hand-holding-usd']"></font-awesome-icon>
                    {{ player.betChip }}
                </div>
                <div class="player__cards">
                    <PokerCard v-for="card in player.cards"
                               :key="String(card.point) + String(card.suit)"
                               :card="card"/>
                </div>
            </div>
        </div>
        <div class="right">
            <div class="info">
                <div class="info__title">Player List</div>
                <div class="info__list">
                    <p v-for="player in room.players"
                       :class="{'info__list--me': me.username === player.username}"
                       :key="player.username">{{ player.username }}</p>
                </div>
                <div class="info__leave" @click="leaveRoom">
                    Leave room
                </div>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import { Player, authorizeSocket, getRoomInfo } from '@/service/api';
import io from 'socket.io-client';
import BASE_URL from '@/service/config';
import PokerCard from '@/components/PokerCard.vue';

interface Card {
    point: number;
    suit: number;
}

interface Options {
    fold: boolean;
    check: boolean;
    call: boolean;
    raise: boolean;
}

interface PlayerInGame extends Player {
    isFold: boolean;
    betChip: number;
    cards: Card[];
}

@Component({
    components: { PokerCard },
})
export default class Room extends Vue {
    socket = io(BASE_URL);

    playerInTurn = '';

    announcement = 'Waiting for players...';

    options: Options = {
        fold: false,
        check: false,
        call: false,
        raise: false,
    };

    pot = 0;

    me: PlayerInGame = {
        username: this.$store.state.username,
        chip: this.$store.state.chip,
        isFold: false,
        betChip: 0,
        cards: [],
    };

    publicCards: Card[] = [];

    room: {
        name: string;
        players: PlayerInGame[];
    } = {
        name: '',
        players: [],
    };

    get otherPlayers() {
        return this.room.players.filter(player => player.username !== this.me.username);
    }

    static extendPlayer(player: Player): PlayerInGame {
        return Object.assign(player, {
            isFold: false,
            betChip: 0,
            cards: [],
        });
    }

    created() {
        this.socket.on('connect', async () => {
            try {
                const { data } = await authorizeSocket(this.socket.id, this.$route.params.roomName);
                if (data.status === 200) {
                    const { data: roomData } = await getRoomInfo(this.$route.params.roomName);
                    if (roomData.status === 200) {
                        this.room.name = roomData.name;
                        this.room.players = roomData.players.map(
                            player => Room.extendPlayer(player),
                        );
                        this.listenEvents();
                    }
                } else {
                    alert('Unauthorized user!');
                    this.$router.push('/dashboard');
                    this.socket.disconnect();
                }
            } catch (e) {
                alert('Server error! Please try again');
                this.$router.push('/dashboard');
            }
        });
    }

    destroyed() {
        this.socket.disconnect();
    }

    hideButtons() {
        this.options = {
            fold: false,
            check: false,
            call: false,
            raise: false,
        };
    }

    leaveRoom() {
        const isSure = window.confirm('Do you really want to leave?\n'
            + '(You will lose all chips you bet if the game has begun)');
        if (isSure) {
            this.$router.push('/dashboard');
        }
    }

    getPlayerByName(username: string): PlayerInGame {
        if (username === this.me.username) {
            return this.me;
        }
        for (let i = 0; i < this.room.players.length; i += 1) {
            if (this.room.players[i].username === username) {
                return this.room.players[i];
            }
        }
        return this.me;
    }

    removePlayerByName(username: string): void {
        for (let i = 0; i < this.room.players.length; i += 1) {
            if (this.room.players[i].username === username) {
                this.room.players.splice(i, 1);
            }
        }
    }

    listenEvents() {
        this.socket.on('join', this.onNewPlayer);
        this.socket.on('publicCard', this.onPublicCard);
        this.socket.on('announcement', this.onAnnouncement);
        this.socket.on('bet', this.onBet);
        this.socket.on('card', this.onCard);
        this.socket.on('myTurn', this.onMyTurn);
        this.socket.on('inTurn', this.onInTurn);
        this.socket.on('addMoney', this.onMoneyAdd);
        this.socket.on('endGame', this.onEndGame);
        this.socket.on('newRound', this.onNewRound);
        this.socket.on('fold', this.onFold);
        this.socket.on('leave', this.onLeftPlayer);
    }

    onNewPlayer(player: Player) {
        this.room.players.push(Room.extendPlayer(player));
    }

    onPublicCard(card: Card) {
        this.publicCards.push(card);
    }

    onAnnouncement(message: string) {
        this.announcement = message;
    }

    onBet(payload: { username: string, chip: number }) {
        const player = this.getPlayerByName(payload.username);
        player.chip -= payload.chip;
        player.betChip += payload.chip;
        this.pot += payload.chip;
    }

    onCard(payload: { username: string, card: Card }) {
        const player = this.getPlayerByName(payload.username);
        if (player.cards.length === 2) {
            return;
        }
        player.cards.push(payload.card);
    }

    onMyTurn(options: Options) {
        this.options = options;
    }

    onInTurn(username: string) {
        if (username !== this.me.username) {
            this.hideButtons();
        }
        this.playerInTurn = username;
    }

    onMoneyAdd(payload: { username: string, chip: number }) {
        const player = this.getPlayerByName(payload.username);
        player.chip += payload.chip;
    }

    onEndGame() {
        this.publicCards = [];
        this.me.cards = [];
        this.me.betChip = 0;
        this.me.isFold = false;
        this.pot = 0;
        this.room.players.forEach((player) => {
            player.cards = [];
            player.betChip = 0;
            player.isFold = false;
        });
        this.announcement = 'Waiting for next game...';
        this.hideButtons();
    }

    onNewRound() {
        this.room.players.forEach((player) => {
            player.betChip = 0;
        });
        this.me.betChip = 0;
    }

    onFold(username: string) {
        const player = this.getPlayerByName(username);
        player.isFold = true;
        player.cards = [];
    }

    onLeftPlayer(username: string) {
        console.log('disconnect');
        this.removePlayerByName(username);
    }

    clickFold() {
        this.socket.emit('fold');
    }

    clickRaise() {
        this.socket.emit('raise');
    }

    clickCall() {
        this.socket.emit('call');
    }

    clickCheck() {
        this.socket.emit('check');
    }
}
</script>

<style lang="scss" scoped>
    @import "../assets/scss/main";

    .room {
        display: flex;
    }

    .left {
        min-width: 980px;
        width: 70vw;
        background-color: $dark-green;
        padding: 100px;
    }

    .right {
        width: 30vw;
        min-width: 420px;
        background-color: $light-green;
    }

    .left, .right {
        position: relative;
        height: 100vh;
        min-height: 800px;
    }

    .room__name {
        font-size: 40px;
        font-weight: bold;
        color: $light-green;
    }

    .info__title {
        font-size: 48px;
        font-weight: bold;
        color: $white;
        margin-left: 100px;
        margin-top: 150px;
    }

    .info__list {
        margin-left: 100px;
        font-size: 30px;
        color: $white;
    }

    .info__list--me {
        color: $red;
    }

    .info__leave {
        @include button;
        position: absolute;
        left: 50%;
        bottom: 100px;
        width: 250px;
        color: $white;
        background-color: $red;
        transform: translateX(-50%);
    }

    .player {
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        font-size: 36px;
        color: $white;
    }

    .player--left {
        left: 100px;
    }

    .player--right {
        right: 75px;
    }

    .player--me {
        position: absolute;
        left: 100px;
        bottom: 100px;
        font-size: 36px;
        color: $red;
    }

    .player--fold {
        color: gray;
    }

    .player--inturn {
        animation: {
            name: twinkle;
            duration: 1.5s;
            direction: alternate;
            iteration-count: infinite;
            timing-function: linear;
        }
    }

    .player__cards {
        display: flex;
        transform: scale(0.5, 0.5);
    }

    .player--left .player__cards {
        justify-content: flex-start;
    }

    .player--right .player__cards {
        justify-content: flex-end;
    }

    .public {
        position: absolute;
        top: 20%;
        left: 50%;
        display: flex;
        transform: translateX(-50%);
    }

    .announcement {
        text-align: center;
        position: absolute;
        font-size: 36px;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        color: $light-green;
    }

    .announcement span {
        margin-left: 10px;
    }

    .panel {
        position: absolute;
        left: 50%;
        transform: translateX(-50%);
        bottom: 100px;
    }

    .panel__cards {
        display: flex;
        justify-content: center;
    }

    .panel__buttons {
        display: flex;
        justify-content: center;
        margin-bottom: 20px;
    }

    .panel__button {
        @include button;
        background-color: $red;
        color: $white;
        width: 120px;
        height: 50px;
        margin: 0 20px;
    }

    .panel__button--green {
        background-color: $light-green;
    }

    @keyframes twinkle {
        from {
            opacity: 1;
        }
        to {
            opacity: 0.2;
        }
    }
</style>
