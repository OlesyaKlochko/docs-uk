# Спеціальні директиви {#custom-directives}

<script setup>
const vFocus = {
  mounted: el => {
    el.focus()
  }
}
</script>

## Знайомство {#introduction}

На додаток до набору директив за замовчуванням, які постачаються в ядрі (наприклад, `v-model` або `v-show`), Vue також дозволяє реєструвати власні користувацькі директиви.

Ми запровадили дві форми повторного використання коду у Vue: [компоненти](/guide/essentials/component-basics.html) і [композиційні функції](./composables). Компоненти є основними будівельними блоками, тоді як композиційні функції зосереджені на повторному використанні логіки стану. Спеціальні директиви, з іншого боку, в основному призначені для повторного використання логіки, яка передбачає низькорівневий доступ DOM до простих елементів.

Спеціальна директива визначається як об'єкт, що містить хуки життєвого циклу, подібні до тих, що є у компоненті. Хуки отримують елемент, до якого прив'язана директива. Ось приклад директиви, яка фокусує введення, коли Vue вставляє елемент у DOM:

<div class="composition-api">

```vue
<script setup>
// вмикає v-focus у шаблонах
const vFocus = {
  mounted: (el) => el.focus()
}
</script>

<template>
  <input v-focus />
</template>
```

</div>

<div class="options-api">

```js
const focus = {
  mounted: (el) => el.focus()
}

export default {
  directives: {
    // вмикає v-focus у шаблоні
    focus
  }
}
```

```vue-html
<input v-focus />
```

</div>

<div class="demo">
  <input v-focus placeholder="Це має бути сфокусованим" />
</div>

Якщо припустити, що ви не клацали ніде на сторінці, введене вище має бути автоматично сфокусованим. Ця директива є більш корисною, ніж атрибут `autofocus`, оскільки вона працює не лише під час завантаження сторінки – вона також працює, коли елемент динамічно вставляється Vue.

<div class="composition-api">

У `<script setup>` будь-яка змінна регістру camelCase, яка починається з префікса `v`, може використовуватися як спеціальна директива. У наведеному вище прикладі `vFocus` можна використовувати в шаблоні як `v-focus`.

Якщо не використовується `<script setup>`, спеціальні директиви можна зареєструвати за допомогою параметра `directives`:

```js
export default {
  setup() {
    /*...*/
  },
  directives: {
    // вмикає v-focus у шаблоні
    focus: {
      /* ... */
    }
  }
}
```

</div>

<div class="options-api">

Подібно до компонентів, спеціальні директиви необхідно зареєструвати, щоб їх можна було використовувати в шаблонах. У наведеному вище прикладі ми використовуємо локальну реєстрацію за допомогою параметра `directives`.

</div>

Також прийнято глобально реєструвати спеціальні директиви на рівні програми:

```js
const app = createApp({})

// робить v-focus придатним для використання в усіх компонентах
app.directive('focus', {
  /* ... */
})
```

:::tip
Спеціальні директиви слід використовувати лише тоді, коли бажана функціональність може бути досягнута лише шляхом прямого маніпулювання DOM. Віддавайте перевагу декларативному шаблону, використовуючи вбудовані директиви, такі як `v-bind`, коли це можливо, тому що вони більш ефективні та зручні для відтворення на сервері.
:::

## Хуки директив {#directive-hooks}

Об'єкт визначення директиви може надавати кілька функцій-хуків (усі необов'язкові):

```js
const myDirective = {
  // викликається перед прив'язкою атрибутів елемента
  // або застосованням слухачів подій
  created(el, binding, vnode, prevVnode) {
    // подробиці аргументів див. нижче
  },
  // викликається безпосередньо перед тим, як елемент буде вставлено в DOM.
  beforeMount(el, binding, vnode, prevVnode) {},
  // викликається, коли зв'язаний елемент є батьківським компонентом
  // і всі його діти змонтовані.
  mounted(el, binding, vnode, prevVnode) {},
  // викликається перед оновленням батьківського компонента
  beforeUpdate(el, binding, vnode, prevVnode) {},
  // викликається після оновлення
  // батьківського компонента й усіх дочірніх елементів
  updated(el, binding, vnode, prevVnode) {},
  // викликається перед тим, як батьківський компонент буде демонтовано
  beforeUnmount(el, binding, vnode, prevVnode) {},
  // викликається, коли батьківський компонент демонтовано
  unmounted(el, binding, vnode, prevVnode) {}
}
```

### Аргументи хуків {#hook-arguments}

Хукам директив передаються такі аргументи:

- `el`: елемент, до якого прив'язана директива. Його можна використовувати для безпосереднього маніпулювання DOM.

- `binding`: об'єкт, що містить наступні властивості:

  - `value`: Значення, передане в директиву. Наприклад, у `v-my-directive="1 + 1"` значенням буде `2`.
  - `oldValue`: Попереднє значення, доступне лише в `beforeUpdate` і `updated`. Він доступний незалежно від того, чи змінилося значення.
  - `arg`: Аргумент передається директиві, якщо така є. Наприклад, у `v-my-directive:foo` аргументом буде `"foo"`.
  - `modifiers`: Об’єкт, що містить модифікатори, якщо такі є. Наприклад, у `v-my-directive.foo.bar` об’єкт-модифікатор буде `{ foo: true, bar: true }`.
  - `instance`: Екземпляр компонента, де використовується директива.
  - `dir`: об'єкт визначення директиви.

- `vnode`: базовий VNode, що представляє зв'язаний елемент.
- `prevNode`: VNode, що представляє прив'язаний елемент із попереднього рендерингу. Доступно лише в хуках `beforeUpdate` і `updated`.

Як приклад розглянемо наступне використання директиви:

```vue-html
<div v-example:foo.bar="baz">
```

Аргумент `binding` буде об'єктом вигляду:

```js
{
  arg: 'foo',
  modifiers: { bar: true },
  value: /* значення `baz` */,
  oldValue: /* значення `baz` з попереднього оновлення */
}
```

Подібно до вбудованих директив, аргументи спеціальних директив можуть бути динамічними. Наприклад:

```vue-html
<div v-example:[arg]="value"></div>
```

Тут аргументи директиви реактивно оновлюватимуться на основі властивості `arg` в стані нашого компонента.

:::tip Примітка
Окрім `el`, вам слід розглядати ці аргументи як лише для читання і ніколи їх не модифікувати. Якщо вам потрібно поширювати інформацію між хуками, рекомендовано це робити використовуючи [dataset](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/dataset) елемента.
:::

## Скорочення функцій {#function-shorthand}

Поширеною для спеціальних директив є та ж сама поведінка, як і для  `mounted` та `updated` без потреби використовувати інші хуки. В таких випадках ми можемо оголосити директиву як функцію:

```vue-html
<div v-color="color"></div>
```

```js
app.directive('color', (el, binding) => {
  // буде викликано для `mounted` та `updated`
  el.style.color = binding.value
})
```

## Об'єктні літерали {#object-literals}

Якщо ваша директива потребує декілька значень, ви також можете передавати літерали об'єктів JavaScript. Запам'ятайте, що директиви можуть приймати будь-які дійсні вирази JavaScript.

```vue-html
<div v-demo="{ color: 'white', text: 'привіт!' }"></div>
```

```js
app.directive('demo', (el, binding) => {
  console.log(binding.value.color) // => "white"
  console.log(binding.value.text) // => "привіт!"
})
```

## Використання з компонентами {#usage-on-components}

При використанні з компонентами, спеціальні директиви завжди будуть застосовуватись до кореневого вузла, подібно до [прохідних атрибутів](/guide/components/attrs.html).

```vue-html
<MyComponent v-demo="test" />
```

```vue-html
<!-- шаблон MyComponent -->

<div> <!-- тут буде застосовано директиву v-demo -->
  <span>Вміст компонента</span>
</div>
```

Зауважте, що компоненти потенційно можуть мати більше, ніж один кореневий вузол. При застовуванні в багатокореневому компоненті, директиву буде проігноровано, про що буде видано попередження. На відміну від атрибутів, директиви не можуть бути передані до іншого елементу за допомогою `v-bind="$attrs"`. Загалом, **не** рекомендовано використовувати спеціальні директиви в компонентах.  
