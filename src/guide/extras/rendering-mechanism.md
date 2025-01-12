---
outline: deep
---

# Механізм рендерингу {#rendering-mechanism}

Як Vue бере шаблон і перетворює його на справжні вузли DOM? Як Vue ефективно оновлює ці вузли DOM? Ми спробуємо освітлити ці питання, занурившись у внутрішній механізм рендерингу Vue.

## Віртуальна DOM {#virtual-dom}

Ви, мабуть, чули про термін "віртуальна DOM", на якому базується система рендерингу Vue.

Віртуальна DOM (VDOM) — це концепція програмування, де ідеальне або "віртуальне" представлення інтерфейсу користувача зберігається в пам'яті та синхронізується з "реальним" DOM. Ця концепція вперше була запроваджена [React](https://reactjs.org/), а потім адаптувана багатьма інших фреймворками із різними реалізаціями, включаючи Vue.

Віртуальна DOM – це більше шаблон, ніж конкретна технологія, тому немає єдиної канонічної реалізації. Ми можемо проілюструвати цю ідею на простому прикладі:

```js
const vnode = {
  type: 'div',
  props: {
    id: 'hello'
  },
  children: [
    /* більше vnodes */
  ]
}
```

Тут `vnode` — це простий об'єкт JavaScript ("віртуальний вузол"), що представляє елемент `<div>`. Він містить всю інформацію, необхідну для створення фактичного елемента. Він також містить більше дочірніх vnodes, що робить його коренем віртуального дерева DOM.

Рендерер під час виконання може проходити віртуальним деревом DOM і створювати з нього реальне дерево DOM. Цей процес називається **монтуванням**.

Якщо у нас є дві копії віртуальних дерев DOM, рендерер також може пройти та порівняти два дерева, з'ясувати відмінності та застосувати ці зміни до фактичного DOM. Цей процес називається **патчем**, також відомий як "розбіжності" або "узгодження".

Основна перевага віртуального DOM полягає в тому, що він дає розробнику можливість програмним шляхом створювати, перевіряти та компонувати бажані структури інтерфейсу користувача декларативним способом, залишаючи безпосередню маніпуляцію DOM рендереру.

## Конвеєр рендерингу {#render-pipeline}

На високому рівні ось що відбувається, коли монтується компонент Vue:

1. **Компіляція**: шаблони Vue скомпільовані у **функції рендереру**: функції, які повертають віртуальні дерева DOM. Цей крок можна виконати або заздалегідь за допомогою етапу збірки, або на льоту за допомогою компіляції під час виконання.

2. **Монтувати**: засіб рендерингу під час виконання викликає функції рендерингу, проходить повернуте віртуальне дерево DOM і створює фактичні вузли DOM на його основі. Цей крок виконується як [реактивний ефект](./reactivity-in-depth), тому він відстежує всі реактивні залежності, які використовувалися.

3. **Патч**: коли змінюється залежність, яка використовується під час монтування, ефект запускається повторно. Цього разу створюється нове оновлене дерево віртуального DOM. Рендерер під час виконання проходить нове дерево, порівнює його зі старим і застосовує необхідні оновлення до фактичного DOM.

![конвеєр рендерингу](./images/render-pipeline.png)

<!-- https://www.figma.com/file/elViLsnxGJ9lsQVsuhwqxM/Rendering-Mechanism -->

## Шаблони чи функцій рендерингу? {#templates-vs-render-functions}

Шаблони Vue скомпільовані у функції рендерингу віртуального DOM. Vue також надає API, які дозволяють нам пропускати етап компіляції шаблону та виконувати безпосередньо авторські функції рендерингу. Функції рендерингу є більш гнучкими, ніж шаблони, коли працюють із високодинамічною логікою, оскільки ви можете працювати з vnodes, використовуючи всю потужність JavaScript.

Отже, чому Vue рекомендує шаблони за замовчуванням? Існує кілька причин:

1. Шаблони ближчі до фактичного HTML. Це полегшує повторне використання існуючих фрагментів HTML, застосування передових методів доступності, створення стилів за допомогою CSS, а також для розуміння та можливості зміни дизайнерами.

2. Шаблони легше аналізувати статично завдяки їх детермінованому синтаксису. Це дозволяє компілятору шаблонів Vue застосовувати багато оптимізацій під час компіляції для покращення продуктивності віртуальної DOM (про що ми розповімо нижче).

На практиці використання шаблонів в застосунках достатньо для більшості випадків. Функції рендерингу зазвичай використовуються лише в повторно використовуваних компонентах, які мають працювати з високодинамічною логікою рендерингу. Використання функції рендерингу обговорюється більш детально в [Функції рендерингу та JSX](./render-function).

## Віртуальна DOM, інформована компілятором {#compiler-informed-virtual-dom}

Віртуальна реалізація DOM у React і більшість інших реалізацій віртуальної DOM є суто в процесі виконання: алгоритм узгодження не може робити жодних припущень щодо вхідного віртуального дерева DOM, тому він має повністю пройти дерево та розрізнити властивості кожного vnode, щоб забезпечити правильність. Крім того, навіть якщо частина дерева ніколи не змінюється, нові vnodes завжди створюються для них під час кожного повторного відтворення, що призводить до непотрібного тиску на пам'ять. Це один із аспектів віртуального DOM, який найбільше критикується: дещо грубий процес узгодження жертвує ефективністю в обмін на декларативність і коректність.

Але це не повинно бути так. Vue фреймворк контролює як компілятор, так і середовище виконання. Це дозволяє нам реалізувати багато оптимізацій під час компіляції, якими може скористатися лише тісно пов'язаний рендерер. Компілятор може статично аналізувати шаблон і залишати підказки в згенерованому коді, щоб середовище виконання могло використовувати їх, коли це можливо. У той же час ми все ще зберігаємо можливість для користувача перейти до рівня функції рендерингу для більш прямого контролю в крайніх випадках. Ми називаємо цей гібридний підхід **Віртуальна DOM, інформована компілятором**.

Нижче ми обговоримо кілька основних оптимізацій, виконаних компілятором шаблонів Vue для покращення продуктивності віртуальної DOM під час виконання.

### Статичне виведення {#static-hoisting}

Досить часто в шаблоні бувають частини, які не містять жодних динамічних прив'язувань:

```vue-html{2-3}
<div>
  <div>foo</div> <!-- виведений -->
  <div>bar</div> <!-- виведений -->
  <div>{{ dynamic }}</div>
</div>
```

[Перегляньте в Template Explorer](https://vue-next-template-explorer.netlify.app/#eyJzcmMiOiI8ZGl2PlxuICA8ZGl2PmZvbzwvZGl2PlxuICA8ZGl2PmJhcjwvZGl2PlxuICA8ZGl2Pnt7IGR5bmFtaWMgfX08L2Rpdj5cbjwvZGl2PiIsInNzciI6ZmFsc2UsIm9wdGlvbnMiOnsiaG9pc3RTdGF0aWMiOnRydWV9fQ==)

Елементи div `foo` і `bar` є статичними - повторне створення vnodes і їх відмінності під час кожного повторного рендерингу не потрібні. Компілятор Vue автоматично виводить виклики створення vnode із функції рендерингу та повторно використовує ті самі vnode під час кожного рендерингу. Засіб візуалізації також може повністю пропустити їх різницю, якщо помітить, що старий vnode і новий vnode є однаковими.

Крім того, коли буде достатньо послідовних статичних елементів, вони будуть згорнуті в один "статичний vnode", який містить простий рядок HTML для всіх цих вузлів ([Приклад](https://vue-next-template-explorer.netlify.app/#eyJzcmMiOiI8ZGl2PlxuICA8ZGl2IGNsYXNzPVwiZm9vXCI+Zm9vPC9kaXY+XG4gIDxkaXYgY2xhc3M9XCJmb29cIj5mb288L2Rpdj5cbiAgPGRpdiBjbGFzcz1cImZvb1wiPmZvbzwvZGl2PlxuICA8ZGl2IGNsYXNzPVwiZm9vXCI+Zm9vPC9kaXY+XG4gIDxkaXYgY2xhc3M9XCJmb29cIj5mb288L2Rpdj5cbiAgPGRpdj57eyBkeW5hbWljIH19PC9kaXY+XG48L2Rpdj4iLCJzc3IiOmZhbHNlLCJvcHRpb25zIjp7ImhvaXN0U3RhdGljIjp0cnVlfX0=)). Ці статичні вузли монтуються прямим налаштуванням `innerHTML`. Вони також кешують свої відповідні вузли DOM під час початкового монтування - якщо той самий фрагмент вмісту повторно використовується в іншому місці програми, нові вузли DOM створюються за допомогою рідного `cloneNode()`, що є надзвичайно ефективним.

### Патч-прапори {#patch-flags}

Для окремого елемента з динамічними зв'язуваннями ми також можемо вивести з нього багато інформації під час компіляції:

```vue-html
<!-- прив'язування лише класу -->
<div :class="{ active }"></div>

<!-- прив'язування лише id та value -->
<input :id="id" :value="value">

<!-- лише текстовий дочірній вузол -->
<div>{{ dynamic }}</div>
```

[Перегляньте в Template Explorer](https://template-explorer.vuejs.org/#eyJzcmMiOiI8ZGl2IDpjbGFzcz1cInsgYWN0aXZlIH1cIj48L2Rpdj5cblxuPGlucHV0IDppZD1cImlkXCIgOnZhbHVlPVwidmFsdWVcIj5cblxuPGRpdj57eyBkeW5hbWljIH19PC9kaXY+Iiwib3B0aW9ucyI6e319)

Під час генерації коду функцій рендерингу для цих елементів Vue кодує тип оновлення, який потребує кожен із них, безпосередньо у виклику створення vnode:

```js{3}
createElementVNode("div", {
  class: _normalizeClass({ active: _ctx.active })
}, null, 2 /* CLASS */)
```

Останній аргумент, `2`, це [патч-прапор](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts). Елемент може мати кілька патч-прапорів, які будуть об'єднані в одне число. Потім рендерер під час виконання може перевірити прапорці за допомогою [побітових операцій](https://en.wikipedia.org/wiki/Bitwise_operation), щоб визначити, чи потрібно йому виконувати певну роботу:

```js
if (vnode.patchFlag & PatchFlags.CLASS /* 2 */) {
  // оновити class елементу
}
```

Побітові перевірки надзвичайно швидкі. Завдяки патч-прапорцям Vue може виконувати найменшу кількість роботи, необхідної під час оновлення елементів із динамічними прив'язуваннями.

Vue також кодує тип дочірніх елементів, які має vnode. Наприклад, шаблон, який має кілька кореневих вузлів, представлений як фрагмент. У більшості випадків ми точно знаємо, що порядок цих кореневих вузлів ніколи не зміниться, тому цю інформацію також можна надати до середовища виконання як патч-прапор:

```js{4}
export function render() {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    /* дочірні елементи */
  ], 64 /* STABLE_FRAGMENT */))
}
```

Таким чином, середовище виконання може повністю пропустити узгодження дочірнього порядку для кореневого фрагмента.

### Зведення дерев {#tree-flattening}

Ще раз поглянувши на згенерований код із попереднього прикладу, ви помітите, що корінь повернутого віртуального дерева DOM створюється за допомогою спеціального виклику `createElementBlock()`:

```js{2}
export function render() {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    /* дочірні елементи */
  ], 64 /* STABLE_FRAGMENT */))
}
```

Концептуально "блок" — це частина шаблону, яка має стабільну внутрішню структуру. У цьому випадку весь шаблон складається з одного блоку, оскільки він не містить жодних структурних директив, таких як `v-if` і `v-for`.

Кожен блок відстежує будь-які вузли-нащадки (не лише прямі дочірні), які мають патч-прапорці. Наприклад:

```vue-html{3,5}
<div> <!-- кореневий блок -->
  <div>...</div>         <!-- не відстежується -->
  <div :id="id"></div>   <!-- відстежується -->
  <div>                  <!-- не відстежується -->
    <div>{{ bar }}</div> <!-- відстежується -->
  </div>
</div>
```

Результатом є зведений масив, який містить лише динамічні вузли-нащадки:

```
div (кореневий блок)
- div із :id прив'язуванням
- div із {{ bar }} прив'язуванням
```

Коли цей компонент потребує повторного рендерингу, йому потрібно лише обійти зведене дерево замість повного дерева. Це називається **зведенням дерева**, і це значно зменшує кількість вузлів, які потрібно пройти під час віртуальної звірки DOM. Будь-які статичні частини шаблону фактично пропускаються.

Директиви `v-if` та `v-for` створять нові вузли блоків:

```vue-html
<div> <!-- кореневий блок -->
  <div>
    <div v-if> <!-- блок if -->
      ...
    <div>
  </div>
</div>
```

Дочірній блок відстежується всередині масиву динамічних нащадків батьківського блоку. Це зберігає стабільну структуру для батьківського блоку.

### Вплив на гідрацію SSR {#impact-on-ssr-hydration}

Патч-прапори та зведення дерева також значно покращують продуктивність [гідрації SSR](/guide/scaling-up/ssr.html#client-hydration) Vue:

- Гідрація одного елемента може здійснюватися швидко на основі патч-прапора відповідного vnode.

- Лише вузли блоків та їхні динамічні нащадки мають бути пройдені під час гідрації, фактично досягаючи часткової гідрації на рівні шаблону.
