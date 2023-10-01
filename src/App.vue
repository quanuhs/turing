<script setup>
    import Line from './components/Line.vue'
    import Rule from './components/Rule.vue'
    import CloseIcon from './components/icons/CloseIcon.vue'
    
    import { ref, reactive, onMounted, computed, watch, defineProps, nextTick } from 'vue'


    const MOVE_LEFT = -1;
    const MOVE_RIGHT = 1;
    const MOVE_STAY = 0;
    
    class RuleClass {
        
        constructor(char, move, stage) {
            this.char = char
            this.move = move
            this.stage = stage
        }

        get(){
            return `${this.char}  ${this.move}  ${this.stage}`
        }
    }

    

    var length = ref(0);
    var input_length = ref(length.value);
    var chars = ref([]);
    var remember_chars = []
    var alphabet = ref(" ");
    var pointer_index = ref(-1);
    var current_index = ref(pointer_index.value)
    var stages = ref({})
    var stage_id = 1
    var rules = ref({})

    
    
    var step_number = 0
    var current_stage = ref(1)
    
    var current_rule = computed(() => {
        return get_rule(chars.value[current_index.value], current_stage.value)
    })

    const line = ref(null)


    
    var ask_before_delete = ref(true)
    var follow_current = ref(true)
    var steps_speed = ref(1)
    var auto_step = ref(false)

    var alphabet_input = ref("")


    const sleep = ms => new Promise(r => setTimeout(r, ms));

    
    watch(length, async (newLength, oldLength) => {
        var result = ref([])
        
        if (pointer_index.value >= length.value){
            pointer_index.value = -1
        }
        
        for (let index = 0; index < length.value; index++) {
            
            if (chars.value[index])
                result.value.push(chars.value[index])
            else
                result.value.push(null)

        }


        chars = result

        
        remember_chars = JSON.parse(JSON.stringify(chars.value));
        
        if (pointer_index.value < 0){
            pointer_index.value = ~~(length.value/2)
        }
        
    })


    watch(pointer_index, async (newValue, oldValue) => {
        current_index.value = newValue
    })

    onMounted( () => {
        length.value = 100

        
        // Загружаем данные из ссылки
        try{
        let hist = window.location.pathname.substring(1)
        let remember = JSON.parse(atob(decodeURIComponent(escape(hist))))

        
        for (let char in remember){
            alphabet_input.value += char
            
        }
        
        saveAlphabet()

        let is_first = true
        for (let char in remember){
            for (let stage in remember[char]){
                if (is_first)
                    addStage()

                rules.value[char][stage].char = remember[char][stage].char
                rules.value[char][stage].move = remember[char][stage].move
                rules.value[char][stage].stage = remember[char][stage].stage
                
            }
            
            is_first = false
            
        }

        // addStage()
        }catch(e){
            saveAlphabet()
            addStage()
        }


        watch(current_index, async (newValue) => {

            if (follow_current.value){
                    await nextTick()
                    line.value.scrollToCurrent(false)
                }
            })


        
    })

    async function reset(){

        if (step_number > 0){
            chars.value = remember_chars
        }
        
        step_number = 0
        current_index.value = pointer_index.value
        current_stage.value = 1
        auto_step.value = false
    }


    async function addStage(){
        stages.value[stage_id] = "Q"+(Object.keys(stages.value).length + 1)
        
        for (let char in rules.value){
            setupRule(char, stage_id)
        }

        stage_id++;
    }

    async function removeStage(stage){
        if (ask_before_delete.value)
            if (!confirm("Вы точно хотите удалить состояние?"))
                return
        
        delete stages.value[stage]
        
        for (let rule in rules.value){
            for (let _stage in rules.value[rule]){
                if (rules.value[rule][_stage].stage == stage)
                    rules.value[rule][_stage].stage = -1
            }
            delete rules.value[rule][stage]
        }

        let _temp_id = 1
        for ( let stage in stages.value){
            stages.value[stage] = "Q"+_temp_id
            _temp_id++
        }

    }


    async function addChar(){
        for (let char of alphabet.value){
            if (rules.value[char] == null){
                rules.value[char] = {}
                
                for (let stage in stages.value){
                    setupRule(char, stage)
                }
            }
            
        }
    }


    watch(rules.value, () => {

        let text = btoa(unescape(encodeURIComponent(JSON.stringify(rules.value))))
        window.history.replaceState(null, "test", text) 
    })

    function setupRule(char, stage) {
        
        rules.value[char][stage] = reactive(new RuleClass(" ", 0, -1))
        
    }

    async function saveAlphabet(){
        
        let oldArray = Array.from(alphabet_input.value)
        let newArray = Array.from(alphabet.value)
        
        let to_add = oldArray.filter(x => !newArray.includes(x));
        let to_remove = newArray.filter(x => !oldArray.includes(x));

        let _temp = new Set(Array.from(alphabet.value + to_add.join("")))
        
        for (let r_char of to_remove){
            if (r_char == " ")
                continue
            
            for (let i = 0; i < length.value; i++){
                if (chars.value[i] == r_char)
                    chars.value[i] = null
            }

            delete rules.value[r_char]
            _temp.delete(r_char)
        }
        
        alphabet.value = Array.from(_temp).join("")
        alphabet_input.value = alphabet.value
        alphabet_input.value = alphabet_input.value.replace(" ", "")

        
        addChar()

        
    }


    async function makeStep(){
        if (auto_step.value == false)
            return
        
        step()
        setTimeout(makeStep, 1000/steps_speed.value)
    }


    async function setCarret(){
        let _text = `число от 0 до ${length.value-1}`
        let carret_index = prompt(`Введите индекс, на который установить каретку (${_text})`)
        
        if (carret_index == null){
            return
        }

        if (0 < carret_index && carret_index <= length.value-1)
            if (/^[1-9]\d*$/.test(carret_index)){
                
                pointer_index.value = parseInt(carret_index)
                return
            }
        
        alert("Укажите положительное число " + _text)
    }

    async function setAlphabet(){
        let _text = `Введите или измените алфавит`
        let new_alphabet = prompt(`${_text}`, alphabet_input.value)
        
        if (new_alphabet == null){
            return
        }

        alphabet_input.value = new_alphabet
        saveAlphabet()
    }

    async function setLength(){
        let _text = `Укажите число - желаемая дина ленты`
        let new_length = prompt(`${_text}`)
        
        if (new_length == null){
            return
        }

        if (0 < new_length)
            if (/^[1-9]\d*$/.test(new_length)){
                
                length.value = new_length
                return
            }
        
        alert("Укажите положительное число! ")
    }




    function get_rule(char, stage) {
        return get_char_rules(char)[stage]
        
    }

    function get_char_rules(char) {
        let char_rules = undefined
    
        if (char === '' || char === null || char === ' ') {
            char_rules = rules.value[" "]
        } else {
    
            char_rules = rules.value[char]
        }

        return char_rules
    }


    async function beforeStep(){
        current_index.value = pointer_index.value
        current_stage.value = 1
        remember_chars = JSON.parse(JSON.stringify(chars.value));
    }
    
    async function step() {
        
        if (step_number == 0) {
            beforeStep()
        }

        let char = chars.value[current_index.value]
    
        let char_rules = get_char_rules(char)
        
        if (char_rules === undefined)
            return 1
    
        let rule = char_rules[current_stage.value]

    
        if (rule === undefined)
            return 2
        
        chars.value[current_index.value] = rule.char
        current_index.value += rule.move
        current_index.value = (chars.value.length + current_index.value) % chars.value.length
        current_stage.value = rule.stage

    
        step_number += 1

        
        
        return chars.value

    }

    
</script>

<template>


  <main>
          <Line :chars="chars" :current="current_index" :allowed_chars="alphabet" ref="line" @changePosition="(index) => pointer_index = index"/>
      
    <div>
        <!-- Начало меню управления -->
        <div class="head-menu">
                <label>Длина ленты: {{length}}</label>
                <label>Позиция каретки: {{current_index}}</label>
                <label>Текущее состояние: <span v-if="current_stage > 0" style="color: orange">Q{{current_stage}}</span><span v-else style="color: red">стоп</span></label>

                <div class="checkline">
                    <label for="ask_remove">Подтверждение удаления</label>
                    <input type="checkbox" id="ask_remove" v-model="ask_before_delete">
                </div>

            <div class="checkline">
                <label for="follow_current">Следить за кареткой</label>
                <input type="checkbox" id="follow_current" v-model="follow_current">
            </div>
                
            
            
        </div>
    
         

        <div class="">

            <div v-if="current_stage > 0" class="button-group">

                
                <button @click="step()" :disabled="!(current_stage > 0)">Шаг</button>
                <button v-if="auto_step" @click="auto_step = false;">Пауза</button>
                <button :disabled="!(current_stage > 0)" v-else @click="auto_step = true; makeStep();">Автоматический шаг</button>

                <div class="slider">
                    <label>Шагов в секунду</label>
                    <div class="oneline">
                        <input type="range" min="1" max="20" step="1" v-model="steps_speed">
                        <label>{{steps_speed}}</label>
                    </div>
                </div>
            </div>

            
            
            <div class="button-group">
                
                <button @click="setLength()">Изменить длину ленты</button>
                <button @click="setCarret()">Установить каретку</button>
                <button @click="setAlphabet()">Изменить алфавит</button>
                    
                <button @click="line.scrollToCurrent()">Выбрать каретку</button>
                <button @click="reset()" :style= "[current_stage <= 0 ? 'color: red' : '']">Сбросить</button>

            </div>


        </div>

        <!-- Конец меню управления -->
    </div>


<div role="region" aria-labelledby="caption" tabindex="0" class="table-rules">
  <table>
    <caption id="caption">Правила
    </caption>
    <thead>
      <tr>
        <th style="height: 100%;"></th>
        <th v-for="(stage, index) in stages">
            {{stage}}
            <a @click.prevent="removeStage(index)" v-if="index > 1" class="stage-delete"><CloseIcon/></a>
        
        </th>
        <th class="">
            <button class="add-stage-button" @click.prevent="addStage()">Добавить состояние</button>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(value, _key) in rules">
        
        <th>{{_key}}</th>
        <td NOWRAP v-for="rule in value" :class="{'selected-rule': rule==current_rule}"><Rule :allowed_chars="alphabet" :rule="rule" :stages="stages"></Rule></td>
        <td></td>
     </tr>

    </tbody>
  </table>
</div>
      
  
  </main>
</template>

<style scoped>

      .button-group{
        display: flex;
        flex-wrap: wrap;
        align-items: stretch;
        justify-content: space-evenly;
        gap: 5px;
        margin-top: 5px;
        margin-bottom: 5px;
    }
    
    .button-group > *{
        flex: 1;
        height: 60px;
        min-width: 120px;
        text-align: center;
    }

    .slider{
        flex: 2;
        border: 2px solid white;
        padding: 5px;
        min-width: 170px;
    }
    
    .slider > div {
        display: flex;
        gap: 5px;
        
    }

    .slider > div > input{
        width: 100%
    }

    .slider > div > label{
        width: 20px;
    }


    .head-menu{
        display: flex;
        justify-content: space-evenly;
        align-items: center;
        font-size: 17px;
        flex-wrap: wrap;
        min-height: 60px;
        border: 2px solid white;
        padding: 10px;
        gap: 5px;
    }

    

    input {
        accent-color: orange;
    }
    
    .alphabet-input{
        letter-spacing: 5px;
        }



    button{
        border: 0px solid orange;
        background-color: transparent;
        color: var(--color-text);
        text-align: center;
        text-decoration: none;
        font-size: 15px;
        font-weight: 600;
        border-radius: 2px;
        cursor: pointer;
        transition: 0.35s;

         &:hover{
            background-color: orange;
            transition: 0.35s;
            color: black;
        }
        
    }
    
    
    .add-stage-button {
        
        height: 100%;
        width: 100%;
        
    }

    .selected-rule{
        padding: 0;
        border: 4px solid orange;
        box-sizing: border-box;

    }

    .stage-delete{
        position: absolute;
        right: 0;
        top: 0;
        height: 60%;
        aspect-ratio: 1/1;
        color: red;
        font-size: 25px;
        display: flex;
        justify-content: center;
        align-items: center;

        &:hover{
            background-color: var(--color-background-mute);
        }
    }

    a{
        cursor: pointer;
    }

    input{
        height: 30px;
    }

    
    
    table {
      font-size: 125%;
      white-space: nowrap;
      margin: 0;
      border: none;
      border-collapse: separate;
      border-spacing: 0;
      table-layout: fixed;
      border: 1px solid black;
    }
    table td,
    table th {
      border: 1px solid black;
      padding: 0.5rem 1rem;
        font-weight: 600;
    }
    
    table thead th {
      padding: 3px;
      position: sticky;
      top: 0;
      z-index: 1;
      min-width: 50px;
      max-width: 180px;
      height: 50px;
    }
    
    table td {
      padding: 4px 5px;
      text-align: center;
    }
    
    table tbody th {
      
      text-align: center;
      position: relative;
      background-color: var(--color-background)
    }
    
    table thead th:first-child {
      position: sticky;
      left: 0;
      z-index: 2;
      padding: 12px;
      background-color: var(--color-background)
    }
    
    table tbody th {
      position: sticky;
      left: 0;
      z-index: 1;
      min-height: 60px;
      height: 60px;
      background-color: var(--color-background)
    }
    caption {
      text-align: left;
      padding: 0.25rem;
      position: sticky;
      left: 0;
    }
    
    [role="region"][aria-labelledby][tabindex] {
      width: 100%;
      max-height: 98vh;
      overflow: auto;
    }
    [role="region"][aria-labelledby][tabindex]:focus {
      /* box-shadow: 0 0 0.5em rgba(0, 0, 0, 0.5); */
      outline: 0;
    }

    .table-rules{
        padding-bottom: 50px;
    }

    .checkline{
        display: flex;
        justify-content: stretch;
        align-items: center;
        gap: 10px;
        align-content: center;
    }

    .checkline > input[type="checkbox"]{
        aspect-ratio: 1/1;
        width: 15px;
        
    }


    
</style>
