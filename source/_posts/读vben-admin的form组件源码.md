---
title: 读vben-admin的form组件源码
tags:
  - JavaScript
abbrlink: b8949dbc
date: 2022-06-28 07:35:54
---

vben-admin 的源码很值得学习！

<!--more-->

### 参数介绍

```javascript
const [register, methods] = useForm(props);
```

**参数 props 内的值可以是 computed 或者 ref 类型**

**register**

`register` 用于注册 `useForm`，如果需要使用 `useForm` 提供的 api，必须将 `register` 传入组件的 `onRegister`

```html
<template>
  <BasicForm @register="register" @submit="handleSubmit" />
</template>
<script>
  export default defineComponent({
    components: { BasicForm },
    setup() {
      const [register, methods] = useForm({
          // ...initConfig 这里传入组件需要的数据
      });
      return {
        register,
      };
    },
  });
</script>
```

这个组件是经过一层组件再次封装的，内部`BasicForm`组件会触发`register`事件，这里在外部捕获到，交由`useForm`内暴露的`register`函数处理。

这样使得`composition`内部可以得到组件实例，从而暴露出各种操作`BasicForm`内部状态的工具函数。

## BasicForm

```javascript
// 导入必要的组件...

export default defineComponent({
    name: 'BasicForm',
    emits: ['register', '...一些事件'],
    setup(props, { emit, attrs }) {
        // 一些组件状态初始化...
        // 该组件实例
        const formElRef = ref<Nullable<FormActionType>>(null);

        // Get the basic configuration of the form
        const getProps = computed((): FormProps => {
        	return { ...props, ...unref(propsRef) } as FormProps;
    	});


    // 省略一些类似工具函数...
    const { handleToggleAdvanced } = useAdvanced({
        advanceState,
        emit,
        getProps,
        getSchema,
        formModel,
        defaultValueRef,
    });


    // 监听自身状态改变...
    watch(
        () => unref(getProps).model,
        () => {
            const { model } = unref(getProps);
            if (!model) return;
            setFieldsValue(model);
        },
        {
            immediate: true,
        },
    );

    // 设置组件Props，这个会被useForm用来设置数据
    async function setProps(formProps: Partial<FormProps>): Promise<void> {
        propsRef.value = deepMerge(unref(propsRef) || {}, formProps);
    }
    
    onMounted(() => {
        initDefault();
        // 触发 register 事件，并携带组件工具函数
        emit('register', formActionType);
    });

    return {
        // ...组件状态和工具函数
      };
     },
});
```

## useForm

```javascript
export function useForm(props?: Props): UseFormReturnType {
  const formRef = ref<Nullable<FormActionType>>(null);
  const loadedRef = ref<Nullable<boolean>>(false);
  // 获取组件暴露的函数
  async function getForm() {
    const form = unref(formRef);
    if (!form) {
      error(
        'The form instance has not been obtained, please make sure that the form has been rendered when performing the form operation!',
      );
    }
    await nextTick();
    return form as FormActionType;
  }
  // 注册处理函数
  function register(instance: FormActionType) {
    isProdMode() &&
      onUnmounted(() => {
        formRef.value = null;
        loadedRef.value = null;
      });
    if (unref(loadedRef) && isProdMode() && instance === unref(formRef)) return;
	// 将事件传入的参数作为ref
    formRef.value = instance;
    loadedRef.value = true;

    watch(
      () => props,
      () => {
        props && instance.setProps(getDynamicProps(props));
      },
      {
        immediate: true,
        deep: true,
      },
    );
  }

  const methods: FormActionType = {
 	// 工具函数
    scrollToField: async (name: NamePath, options?: ScrollOptions | undefined) => {
      const form = await getForm();
      form.scrollToField(name, options);
    },
    setProps: async (formProps: Partial<FormProps>) => {
      const form = await getForm();
      form.setProps(formProps);
    },

    updateSchema: async (data: Partial<FormSchema> | Partial<FormSchema>[]) => {
      const form = await getForm();
      form.updateSchema(data);
    },

    resetSchema: async (data: Partial<FormSchema> | Partial<FormSchema>[]) => {
      const form = await getForm();
      form.resetSchema(data);
    },

    clearValidate: async (name?: string | string[]) => {
      const form = await getForm();
      form.clearValidate(name);
    },

    resetFields: async () => {
      getForm().then(async (form) => {
        await form.resetFields();
      });
    },

    removeSchemaByFiled: async (field: string | string[]) => {
      unref(formRef)?.removeSchemaByFiled(field);
    },

    // TODO promisify
    getFieldsValue: <T>() => {
      return unref(formRef)?.getFieldsValue() as T;
    },

    setFieldsValue: async <T>(values: T) => {
      const form = await getForm();
      form.setFieldsValue<T>(values);
    },

    appendSchemaByField: async (
      schema: FormSchema,
      prefixField: string | undefined,
      first: boolean,
    ) => {
      const form = await getForm();
      form.appendSchemaByField(schema, prefixField, first);
    },

    submit: async (): Promise<any> => {
      const form = await getForm();
      return form.submit();
    },

    validate: async (nameList?: NamePath[]): Promise<Recordable> => {
      const form = await getForm();
      return form.validate(nameList);
    },

    validateFields: async (nameList?: NamePath[]): Promise<Recordable> => {
      const form = await getForm();
      return form.validateFields(nameList);
    },
  };

  return [register, methods];
}
```



## 总结

之前我一般是传入`组件ref`到`useForm`，对于想要使用组件内部函数，通常是传入回调函数或者直接操作组件，管理起来异常复杂。

不如这样写，不仅可以传入组件ref，还能传入其他组件ref，如 `BasicTable`组件触发 `register` 事件：

```javascript
emit('register', tableAction, formActions);
```

留下当时看源码画的图，希望能找到更符合描述事件驱动的程序源码的图表。

![](./读vben-admin的form组件源码/show.png)

[vbenjs/vue-vben-admin: A modern vue admin. It is based on Vue3, vite and TypeScript. It's fast！ (github.com)](https://github.com/vbenjs/vue-vben-admin)
