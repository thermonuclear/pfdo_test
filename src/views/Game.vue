<template lang="pug">
  .about
    h1.loading(v-if='loading') Идет загрузка карточки...
    h1.warning(v-else-if='!word') Ваш словарь пуст либо все слова разыграны.
    .card(v-else)
      .content
        h3.placeholder Собери слово из букв
        .result
          .result-letter(v-for="item in wordSplit"
            v-bind:class="{'current-letter': item.current, 'is-answered': item.show}")
            span(v-show="item.show") {{ item.letter }}
        .wrapper-letters
          .letters
            .letter(v-for="(s, index) in symbolShuffle"
              v-bind:class="{error: s.error, hidden: !s.show}"
              @click="clickLetter(s.letter, index)")
              .letter-content
                span {{ s.letter }}
                span.count(v-if="s.count > 1") {{ s.count }}
        .message(v-show="isWin")
          div Вы выиграли! Повторить?
          .yes(@click="initCard") Конечно!
          .no(@click="total") Я уже устал
    .total(v-if="showTotal")
      SortedTable(:values="usedIds" ascIcon="<span> ▲</span>" descIcon="<span> ▼</span>")
        thead
          tr
            th(scope="col")
              SortLink(name="index") №
            th(scope="col")
              SortLink(name="word") Слово
            th(scope="col")
              SortLink(name="deltaTime") Время, сек.
        tbody(slot="body" slot-scope="sort")
          tr(v-for="value in sort.values" :key="value.index")
            td {{ value.index }}
            td {{ value.word }}
            td {{ value.deltaTime }}
</template>

<script>
import { SortedTable, SortLink } from 'vue-sorted-table';

export default {
  name: 'Game',
  components: {
    SortedTable,
    SortLink,
  },
  data: () => ({
    loading: true,
    word: null,
    // кол-во слов в словаре
    startId: 2,
    endId: 1368,
    // кол-во использованных слов
    usedWords: 0,
    freeIds: [],
    usedIds: [],
    wordSplit: [],
    symbolShuffle: [],
    // текущий индекс угадываемой буквы
    wordSplitIndex: 0,
    isWin: false,
    showTotal: false,
  }),
  mounted() {
    for (let i = this.startId; i <= this.endId; i += 1) {
      this.freeIds.push(i);
    }

    this.initCard();
  },
  methods: {
    // вывод игровой карточки
    async initCard() {
      this.loading = true;
      this.word = null;
      this.isWin = false;
      this.showTotal = false;
      this.wordSplitIndex = 0;
      this.wordSplit = [];
      this.symbolShuffle = [];
      await this.getWord();
      // если слово не нашли
      if (!this.word) return;

      const symbolCount = {};
      this.word.split('').forEach((item) => {
        if (symbolCount[item] === undefined) {
          symbolCount[item] = 1;
        } else {
          symbolCount[item] += 1;
        }
        this.wordSplit.push({ letter: item, show: false, current: false });
      });
      this.wordSplit[0].current = true;

      this.shuffle(Object.keys(symbolCount)).forEach((letter) => {
        this.symbolShuffle.push({
          letter,
          count: symbolCount[letter],
          show: true,
          error: false,
        });
      });

      this.wordSplitIndex = 0;
      if (this.usedIds.length) {
        this.usedIds[this.usedIds.length - 1].startTime = Date.now();
      }

      this.loading = false;
    },
    // получение слова
    async getWord() {
      // eslint-disable-next-line
      while (true) {
        let isActiveWord = false;
        if (!this.freeIds.length) break;

        const index = Math.round(Math.random() * (this.freeIds.length - 1));
        const id = this.freeIds[index];

        // eslint-disable-next-line
        await this.axios.get(`https://apidir.pfdo.ru/v1/directory-program-activities/${id}`).then((response) => {
          const { data } = response.data;
          // Если слово есть и статус его активный
          if (data !== undefined && Number(data.status) === 10) {
            isActiveWord = true;
            const word = data.name.toUpperCase();
            // список слов, выбранных для угадывания
            this.usedIds.push({ id, word, index: this.usedIds.length + 1 });
            this.word = word;
          }
        });
        // убираем id слова из доступных для последующего выбора
        this.freeIds.splice(index, 1);
        if (isActiveWord) break;
      }
    },
    // Смешиваем рандомно порядок элементов массива  https://learn.javascript.ru/task/shuffle (Тасование Фишера — Йетса)
    shuffle(array) {
      for (let i = array.length - 1; i > 0; i -= 1) {
        const j = Math.floor(Math.random() * (i + 1));
        // eslint-disable-next-line
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    // клик по букве в блоке выбора букв
    clickLetter(letter, index) {
      // символ совпадает
      if (letter === this.wordSplit[this.wordSplitIndex].letter) {
        // считаем кол-во оставшихся повторов буквы
        this.symbolShuffle[index].count -= 1;
        // скрываем букву, если ее больше нет в оставшейся части слова
        if (this.symbolShuffle[index].count === 0) {
          this.symbolShuffle[index].show = false;
        }

        // показываем угаданную букву и убираем статус текущей
        this.wordSplit[this.wordSplitIndex].show = true;
        this.wordSplit[this.wordSplitIndex].current = false;
        // помечаем следующую букву слова для отгадывания
        this.wordSplitIndex += 1;
        // еще есть неотгаданные буквы в слове
        if (this.wordSplit[this.wordSplitIndex] !== undefined) {
          this.wordSplit[this.wordSplitIndex].current = true;
        } else {
          // отгадали все буквы
          this.isWin = true;
          const endTime = Date.now();
          const lastIndex = this.usedIds.length - 1;
          this.usedIds[lastIndex].endTime = endTime;
          // продолжительность отгадывания слова в сек.
          this.usedIds[lastIndex]
            .deltaTime = Math.round((endTime - this.usedIds[lastIndex].startTime) / 1000);
        }
      } else {
        // символ не совпадает, подсвечиваем ошибку
        this.symbolShuffle[index].error = true;
        setTimeout(() => {
          this.symbolShuffle[index].error = false;
        }, 300);
      }
    },
    // показ итоговой таблицы
    total() {
      this.showTotal = true;
    },
  },
};
</script>

<style lang="scss">
  .warning {
    color: yellow;
  }
  .loading {
    color: #42b983;
  }
  .card, .total {
    max-width: 760px;
    margin: 0 auto;
    border: 1px solid #e6e9ee;
    box-sizing: border-box;
  }
  .total {
    margin-top: 20px;
    border: none;
    table {
      width: 100%;
      max-width: 100%;
      overflow-x: auto;
      margin: .5em 0 2.5em;
      border-spacing: 0;
      border-collapse: collapse;
      vertical-align: top;
      font-size: 1.6rem;
      td, th {
        padding: 6px 12px;
        border: 1px solid #e3ecf3;
      }
      th {
        color: #15171a;
        font-size: 1.2rem;
        font-weight: 700;
        letter-spacing: .2px;
        /*text-align: left;*/
        text-transform: uppercase;
        background-color: #FAFAFA;
      };
    }
  }
  .content {
    padding: 28px 28px 24px;
    position: relative;
  }
  .placeholder {
    color: #7e919f;
  }
  .result {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    flex-flow: wrap;
    padding: 0;
    word-break: break-all;
    text-align: center;
  }
  .result-letter {
    display: inline-block;
    border-radius: 4px;
    box-shadow: inset 0 1px 3px 0 #eaebec;
    background-color: #f5f7f8;
    width: 62px;
    height: 50px;
    margin: 6px;
    color: #fff;
    font-size: 20px;
    font-weight: 600;
    line-height: 47px;
    box-sizing: border-box;
  }
  .current-letter {
    border: 2px solid rgba(37,130,231,.6);
  }
  .is-answered {
    opacity: 1;
    background-color: #28c38a;
    box-shadow: none;
  }
  .wrapper-letters {
    box-sizing: border-box;
  }
  .letters {
    padding: 18px 11px 14px;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: center;
    .hidden {
      visibility: hidden;
    }
  }
  .letter {
    width: 62px;
    height: 50px;
    margin: 6px;
    padding: 0;
    color: #fff;
    font-size: 18px;
    font-weight: 600;
    line-height: 30px;
    background-color: #2582e7;
    white-space: nowrap;
    cursor: pointer;
    border-radius: 4px;
    /*transition: all .3s ease-out;*/
    /*text-transform: none;*/
    align-self: center;
  }
  .error {
    background-color: red;
  }
  .letter-content {
    display: inline-flex;
    justify-content: center;
    align-items: center;
    white-space: nowrap;
    line-height: 1.43;
    position: relative;
    span:first-of-type {
      display: inline-block;
      position: relative;
      top: 9px;
      height: 19px;
      box-sizing: border-box;
    }
    .count {
      position: absolute;
      top: -12px;
      right: -34px;
      width: 17px;
      height: 17px;
      color: #fff;
      font-size: 11px;
      font-weight: 600;
      border-radius: 50%;
      border: 1px solid #fff;
      text-align: center;
      line-height: 16px;
      background-color: #2582e7;
    }
  }
  .message {
    margin-top: 10px;
    font-size: 18px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    box-sizing: border-box;
    background-color: #f5f7f8;
    div {
      margin: 10px;
    }
    .yes, .no {
      border-radius: 2px;
      padding: 6px;
      color: #fff;
      cursor: pointer;
    }
    .yes {
      background-color: #28c38a;
    }
    .no {
      background-color: orange;
    }
  }
</style>
