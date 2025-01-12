# Специфікація синтаксису одно-файлових компонент {#sfc-syntax-specification}

## Огляд {#overview}

Одно-файловий компонент Vue (SFC), який традиційно використовує розширення файлу `*.vue`, є спеціальним форматом файлу, який використовує HTML-подібний синтаксис для опису компонента Vue. Одно-файловий компонент Vue синтаксично сумісний з HTML.

Кожен файл `*.vue` складається з трьох типів мовних блоків верхнього рівня: `<template>`, `<script>` і `<style>`, а також додаткових користувацьких блоків:

```vue
<template>
  <div class="example">{{ msg }}</div>
</template>

<script>
export default {
  data() {
    return {
      msg: 'Hello world!'
    }
  }
}
</script>

<style>
.example {
  color: red;
}
</style>

<custom1>
  Це може бути напр. документація для компонента.
</custom1>
```

## Мовні блоки {#language-blocks}

### `<template>` {#template}

- Кожен файл `*.vue` може містити не більше одного блоку `<template>` верхнього рівня.

- Вміст буде витягнуто та передано до `@vue/compiler-dom`, де попередньо скомпілюється в JavaScript функцію рендерингу і приєднано до компонента, що експортується, як його опція `render`.

### `<script>` {#script}

- Кожен файл `*.vue` може містити не більше одного блоку `<script>` (за винятком [`<script setup>`](/api/sfc-script-setup.html)).

- Сценарій виконується як ES модуль.

- **Експорт за промовчанням** повинен бути об'єктом опцій компонента Vue, або звичайним об'єктом, або як значення, що повертається [defineComponent](/api/general.html#definecomponent).

### `<script setup>` {#script-setup}

- Кожен файл `*.vue` може містити не більше одного блоку `<script setup>` (за винятком звичайного `<script>`).

- Сценарій попередньо обробляється і використовується як функція компонента `setup()`, що означає, що вона виконуватиметься **для кожного екземпляра компонента**. Прив'язки верхнього рівня `<script setup>` автоматично оголошуються в шаблоні. Більш детальну інформацію можна знайти [на спеціальній сторінці документації `<script setup>`](/api/sfc-script-setup)

### `<style>` {#style}

- Один файл `*.vue` може містити кілька тегів `<style>`.

- Тег `<style>` може мати атрибути `scoped` або `module` (додаткову інформацію див. у розділі [можливості стилів одно-файлового компонента](/api/sfc-css-features), щоб допомогти інкапсулювати стилі в поточний компонент. В одному компоненті можна змішувати кілька тегів `<style>` з різними режимами інкапсуляції.

### Користувацькі блоки {#custom-blocks}

Додаткові користувацькі блоки можна включити у файл `*.vue` для будь-яких потреб конкретного проекту, наприклад, блок `<docs>`. Деякі реальні приклади спеціальних блоків включають:

- [Gridsome: `<page-query>`](https://gridsome.org/docs/querying-data/)
- [vite-plugin-vue-gql: `<gql>`](https://github.com/wheatjs/vite-plugin-vue-gql)
- [vue-i18n: `<i18n>`](https://github.com/intlify/bundle-tools/tree/main/packages/vite-plugin-vue-i18n#i18n-custom-block)

Обробка користувацьких блоків залежатиме від інструментів. Якщо ви хочете створити власні користувацькі інтеграції блоків, див. [відповідний розділ інструментів](/guide/scaling-up/tooling.html#sfc-custom-block-integrations), щоб отримати докладнішу інформацію.

## Автоматичне визначення імені {#automatic-name-inference}

Одно-файлові компоненти автоматично визначають ім'я компонента на **ім'я файлу** в таких випадках:

- Форматування попереджень під час розробки
- Інспектування у DevTools
- Рекурсивне посилання на самого себе. Наприклад, файл з ім'ям `FooBar.vue` може посилатися на себе як `<FooBar/>` у своєму шаблоні. Це має нижчий пріоритет, ніж явно зареєстровані/імпортовані компоненти.

## Пре-процесори {#pre-processors}

У блоках можна оголосити мову пре-процесора за допомогою атрибуту `lang`. Найпоширенішим випадком є використання TypeScript для блоку `<script>`:

```vue-html
<script lang="ts">
  // використовуємо TypeScript
</script>
```

`lang` можна застосувати до будь-якого блоку - наприклад, ми можемо використовувати `<style>` з [Sass](https://sass-lang.com/) і `<template>` з [Pug](https:/ /pugjs.org/api/getting-started.html):

```vue-html
<template lang="pug">
p {{ msg }}
</template>

<style lang="scss">
  $primary-color: #333;
  body {
    color: $primary-color;
  }
</style>
```

Зауважте, що інтеграція з різними пре-процесорами може відрізнятися залежно від ланцюжка інструментів. Перегляньте відповідну документацію для прикладів:

- [Vite](https://vitejs.dev/guide/features.html#css-pre-processors)
- [Vue CLI](https://cli.vuejs.org/guide/css.html#pre-processors)
- [webpack + vue-loader](https://vue-loader.vuejs.org/guide/pre-processors.html#using-pre-processors)

## Імпорти через src {#src-imports}

Якщо ви віддаєте перевагу розділенню ваших компонентів `*.vue` на кілька файлів, ви можете використовувати атрибут `src`, щоб імпортувати зовнішній файл для мовного блоку:

```vue
<template src="./template.html"></template>
<style src="./style.css"></style>
<script src="./script.js"></script>
```

Пам’ятайте, що імпорт `src` дотримується тих самих правил вирішення шляху, що й запити модуля webpack, що означає:

- Відносні шляхи повинні починатися з `./`
- Ви можете імпортувати ресурси із залежностей npm:

```vue
<!-- імпорт файлу із встановленого npm-пакету "todomvc-app-css" -->
<style src="todomvc-app-css/index.css" />
```

`src` імпорт також працює з користувацькими блоками, напр.:

```vue
<unit-test src="./unit-test.js">
</unit-test>
```

## Коментарі {#comments}

У кожному блоці слід використовувати синтаксис коментарів мови (HTML, CSS, JavaScript, Pug, і т.д.). Для коментарів верхнього рівня слід використовувати синтаксис HTML-коментарів:: `<!-- comment contents here -->`
