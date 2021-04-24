---
title: Лекции по фронтенду - Тестируем Redux Thunk
---

## Тестируем Redux Thunk

![react-testing library](assets/redux-api/redux-thunk-meme.png)

[Дмитрий Вайнер](https://github.com/dmitryweiner)

[видео](https://drive.google.com/file/d/1gGJaIE0CzD7yv8Ya0T7wayXgdI3ZMYmz/view?usp=sharing)

---

### Методика
* Создаём поддельный стор, в котором мы сможем:
  * Наблюдать вызов экшенов.
  * Отслеживать состояние.
* Пишем моки на API, чтобы не ходить в сеть.
* Вызываем асинхронный экшен (напрямую или в связанном компоненте).
* Проверяем, что вызванные экшены соответствуют желаемым.
* Логику редьюсеров тестируем отдельно.
  
---

### Поддельный стор
* Используем библиотеку [redux-mock-store](https://github.com/reduxjs/redux-mock-store).
```shell
npm i -D redux-mock-store
npm i -D @types/redux-mock-store # типы для TS
```
* Пример использования:

```js
import thunkMiddleware from 'redux-thunk';
import configureStore from 'redux-mock-store';
const middlewares = [thunkMiddleware];
const mockStore = configureStore(middlewares);
// в тесте
const initialState = {}
const store = mockStore(initialState);
await store.dispatch(addTodo()); // Dispatch the action
const actions = store.getActions(); // Test if your store dispatched the expected actions
const expectedPayload = { type: 'ADD_TODO' };
expect(actions).toEqual([expectedPayload]);
```

---

### Модифицируем ```setupTests.js```

```js
import '@testing-library/jest-dom';
import { render } from '@testing-library/react';
import { createStore } from 'redux';
import { Provider } from 'react-redux';
import { reducer, initialState as originalInitialState } from './store';
import thunkMiddleware from 'redux-thunk';
import configureStore from 'redux-mock-store';
//
const middlewares = [thunkMiddleware];
const mockStore = configureStore(middlewares); // функция для создания стора
//
const TestProvider = ({ store, children }) => <Provider store={store}>{children}</Provider>;
//
export function testRender(ui, { store, ...otherOpts }) {
    return render(<TestProvider store={store}>{ui}</TestProvider>, otherOpts)
}
//
export function makeTestStore({ initialState = originalInitialState, useMockStore = false } = {}) {
    let store;
    if (useMockStore) {
        store = mockStore(initialState); // развилка
    } else {
        store = createStore(reducer, initialState)
    }
    const origDispatch = store.dispatch;
    store.dispatch = jest.fn(origDispatch);
    return store;
}
```

---

### Моки API
* Используем библиотеку ```fetch-mock```:
```shell
npm i -D fetch-mock
```
* Пример использования:
```js
afterEach(() => fetchMock.reset()); // не забыть сбросить состояние
//
fetchMock.mock(
    'URL', // какой адрес перехватывать
    {
        status: 200, // код ответа
        body: { id: 123 } // ответ сервера
    }, {
        method: 'POST', body: { title: 123 } // что ожидается в теле запроса
    }
);
```
* [Шпаргалка](https://github.com/wheresrhys/fetch-mock/blob/master/docs/cheatsheet.md),
  [документация (хорошая)](http://www.wheresrhys.co.uk/fetch-mock/).

---

### Асинхронный экшен
```js
export const addElement = (title: string) => async (dispatch: AppDispatch) => {
    try {
        dispatch(setRequestStatus(REQUEST_STATUS.LOADING));
        const data = await api.todos.add({title});
        dispatch({type: ACTION_TYPES.ADD, payload: data});
        dispatch(setRequestStatus(REQUEST_STATUS.SUCCESS));
    } catch (error) {
        dispatch(setError(error.message));
        dispatch(setRequestStatus(REQUEST_STATUS.ERROR));
    }
}
```

---

### Тест асинхронного экшена
Не забыть про ```async/await```!
```js
afterEach(() => fetchMock.reset());
//
test('тестирование асинхронного экшена', async () => {
    const title = 'test';
    const element = {
        id: '123',
        title,
        isChecked: false
    };
    fetchMock.mock(
        'express:/todos',
        {
            status: 200,
            body: element
        }, {
            method: 'POST'
        }
    );
    const store = makeTestStore({ useMockStore: true });
    await store.dispatch(addElement(title)); // без await не работает
    expect(store.getActions()).toEqual([
        {type: ACTION_TYPES.SET_REQUEST_STATUS, payload: REQUEST_STATUS.LOADING},
        {type: ACTION_TYPES.ADD, payload: element},
        {type: ACTION_TYPES.SET_REQUEST_STATUS, payload: REQUEST_STATUS.SUCCESS}
    ])
});
```

---

### Тестирование компонента
* Инициируем действие, вызывающее запуск асинхронного экшена.
* Проверяем, что компонент вызовет асинхронный экшен:
  * В массиве ```store.getActions()``` должен быть соответствующий экшен начала действия.
* Что сам асинхронный экшен работает, мы проверили в предыдущей части.

---

### Тестирование формы
* Допустим форма вызывает экшен при отправке:
```ts
function innerSubmit(e: FormEvent) {
    e.preventDefault();
    dispatch(addElement(value));
}
```
* Проверим, что экшен начал работу:
```js
expect(store.dispatch).not.toBeCalled();
fireEvent.submit(screen.getByTestId('form')); // отправка формы
expect(store.getActions()[0]).toEqual({
    type: ACTION_TYPES.SET_REQUEST_STATUS,
    payload: REQUEST_STATUS.LOADING
});
```

---

### Полезные ссылки
* [Документация по redux-mock-store](https://github.com/reduxjs/redux-mock-store).
* [Использование fetch-mock (по документации Redux)](https://redux.js.org/recipes/writing-tests#async-action-creators).
* [Шпаргалка по fetch-mock](https://github.com/wheresrhys/fetch-mock/blob/master/docs/cheatsheet.md).
* [Документация по fetch-mock](http://www.wheresrhys.co.uk/fetch-mock/).
* [Пространные рассуждения о том, как писать тесты](https://michalzalecki.com/testing-redux-thunk-like-you-always-want-it/).