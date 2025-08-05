# Datart 前端代码编写规范

## 1. 项目架构概述

### 1.1 技术栈
- **框架**: React 17.0.2 + TypeScript 4.5.4
- **状态管理**: Redux Toolkit + React Redux
- **UI组件库**: Ant Design 4.16.13
- **样式方案**: Styled Components 5.3.3
- **构建工具**: Create React App + Craco
- **国际化**: i18next + react-i18next
- **图表库**: ECharts 5.3.1 + @antv/s2

### 1.2 项目结构
```
src/
├── app/                    # 应用主体
│   ├── components/         # 通用组件
│   ├── contexts/          # React Context
│   ├── hooks/             # 自定义Hooks
│   ├── pages/             # 页面组件
│   ├── slice/             # Redux状态切片
│   ├── types/             # 类型定义
│   └── utils/             # 工具函数
├── locales/               # 国际化资源
├── redux/                 # Redux配置
├── styles/                # 全局样式
├── types/                 # 全局类型定义
└── utils/                 # 全局工具函数
```

## 2. TypeScript 编写规范

### 2.1 类型定义规范
```typescript
// 接口定义使用 PascalCase
export interface User {
  id: string;
  username: string;
  email: string;
  avatar: string;
  name: string | null;
  description: string;
  orgOwner?: boolean;
  timezone?: string;
}

// 使用枚举替代字符串联合类型（重要更新）
export enum MetricStatus {
  DRAFT = 'DRAFT',
  PUBLISHED = 'PUBLISHED',
  ARCHIVED = 'ARCHIVED',
}

export enum DataType {
  INTEGER = 'INTEGER',
  DECIMAL = 'DECIMAL',
  STRING = 'STRING',
  DATE = 'DATE',
  DATETIME = 'DATETIME',
}

// 泛型接口定义
export interface APIResponse<T> {
  success: boolean;
  data: T;
  message?: string;
}

// 分页响应类型定义（新增）
export interface PageResponse<T> {
  list: T[];
  total: number;
  page: number;
  size: number;
}
```

### 2.2 组件Props类型定义
```typescript
interface LoginFormProps {
  loading: boolean;
  loggedInUser?: User | null;
  oauth2Clients: Array<{ name: string; value: string }>;
  registerEnable?: boolean;
  inShare?: boolean;
  onLogin?: (value) => void;
}

// 使用泛型的组件Props
interface ItemLayoutProps<T> {
  ancestors: string[];
  translate?: (key: string, disablePrefix?: boolean) => string;
  data: T;
  onChange?: (ancestors: string[], value: any) => void;
}
```

### 2.3 Hooks类型定义
```typescript
// 自定义Hook返回类型
function usePrefixI18N(prefix?: string): (
  key: string, 
  disablePrefix?: boolean, 
  options?: any
) => string {
  // Hook实现
}

// 异步操作参数类型
export interface LoginParams {
  params: {
    username: string;
    password: string;
  };
  resolve: () => void;
}
```

## 3. React 组件编写规范

### 3.1 函数组件定义
```typescript
import React, { FC, memo, useCallback, useState } from 'react';

// 使用 FC 类型和 memo 优化
export const LoginForm: FC<LoginFormProps> = memo(({
  loading,
  loggedInUser,
  oauth2Clients,
  registerEnable = true,
  inShare = false,
  onLogin,
}) => {
  // 组件实现
});

// 或者使用函数声明
export function LoginForm({
  loading,
  loggedInUser,
  // ...其他props
}: LoginFormProps) {
  // 组件实现
}
```

### 3.2 Hooks使用规范
```typescript
export function LoginForm({ onLogin }: LoginFormProps) {
  // 状态定义
  const [switchUser, setSwitchUser] = useState(false);
  const [form] = Form.useForm();
  
  // 路由和国际化
  const history = useHistory();
  const t = usePrefixI18N('login');
  const tg = usePrefixI18N('global');
  
  // 计算属性
  const logged = useMemo(() => !!getToken(), []);
  
  // 事件处理函数使用 useCallback
  const toApp = useCallback(() => {
    history.replace('/');
  }, [history]);

  const onSwitch = useCallback(() => {
    setSwitchUser(true);
  }, []);

  const toAuthClient = useCallback(
    clientUrl => () => {
      if (inShare) {
        persistence.session.save(
          StorageKeys.AuthRedirectUrl,
          window.location.href,
        );
      }
      window.location.href = clientUrl;
    },
    [inShare],
  );

  return (
    // JSX内容
  );
}
```

### 3.3 组件性能优化
```typescript
// 使用 memo 和自定义比较函数
const BasicColorSelector: FC<ItemLayoutProps<ChartStyleConfig>> = memo(
  ({ ancestors, translate: t = title => title, data: row, onChange }) => {
    // 组件实现
  },
  itemLayoutComparer, // 自定义比较函数
);

// 比较函数实现
export const itemLayoutComparer = (
  prevProps: ItemLayoutProps<any>,
  nextProps: ItemLayoutProps<any>
) => {
  return (
    prevProps.data === nextProps.data &&
    prevProps.ancestors === nextProps.ancestors
  );
};
```

## 4. Redux状态管理规范

### 4.1 Slice定义规范
```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

// 初始状态定义
export const initialState: AppState = {
  loggedInUser: null,
  systemInfo: null,
  setupLoading: false,
  loginLoading: false,
  registerLoading: false,
  saveProfileLoading: false,
  modifyPasswordLoading: false,
  oauth2Clients: [],
};

// Slice创建
const slice = createSlice({
  name: 'app',
  initialState,
  reducers: {
    updateUser(state, { payload }: PayloadAction<User>) {
      state.loggedInUser = payload;
    },
  },
  extraReducers: builder => {
    // 异步操作处理
    builder.addCase(login.pending, state => {
      state.loginLoading = true;
    });
    builder.addCase(login.fulfilled, (state, action) => {
      state.loginLoading = false;
      state.loggedInUser = action.payload;
    });
    builder.addCase(login.rejected, state => {
      state.loginLoading = false;
    });
  },
});
```

### 4.2 异步Thunk定义
```typescript
import { createAsyncThunk } from '@reduxjs/toolkit';

export const login = createAsyncThunk<User, LoginParams>(
  'app/login',
  async ({ params, resolve }) => {
    const { data } = await request2<User>({
      url: '/users/login',
      method: 'POST',
      data: params,
    });
    localStorage.setItem(StorageKeys.LoggedInUser, JSON.stringify(data));
    resolve();
    return data;
  },
);

// 带错误处理的异步操作
export const getUserInfoByToken = createAsyncThunk<
  User | undefined,
  UserInfoByTokenParams
>('app/getUserInfoByToken', async ({ token, resolve, reject }) => {
  setToken(token);
  const { data } = await request2<User>(
    {
      url: '/users',
      method: 'GET',
    },
    undefined,
    {
      onRejected(error) {
        reject();
        removeToken();
      },
    },
  );
  localStorage.setItem(StorageKeys.LoggedInUser, JSON.stringify(data));
  resolve();
  return data;
});
```

### 4.3 Slice使用规范
```typescript
// 在组件中使用
export const useAppSlice = () => {
  useInjectReducer({ key: slice.name, reducer: slice.reducer });
  return { actions: slice.actions };
};

// 组件中调用
const dispatch = useDispatch();
const { actions } = useAppSlice();

const handleLogin = useCallback((values) => {
  dispatch(login({
    params: values,
    resolve: () => {
      // 成功回调
    }
  }));
}, [dispatch]);
```

## 5. API请求规范

### 5.1 请求工具配置
```typescript
import axios, { AxiosRequestConfig, AxiosResponse } from 'axios';

// 创建axios实例
export const instance = axios.create({
  baseURL: BASE_API_URL,
  validateStatus(status) {
    return status < 400;
  },
});

// 请求拦截器
instance.interceptors.request.use(config => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = token;
  }
  return config;
});

// 响应拦截器
instance.interceptors.response.use(response => {
  // 刷新访问令牌
  const token = response.headers.authorization;
  if (token) {
    setToken(token);
  }
  return response;
});
```

### 5.2 统一请求方法
```typescript
/**
 * 统一HTTP请求工具
 * @template T 响应数据类型
 * @param {string | AxiosRequestConfig} url 请求URL或配置
 * @param {AxiosRequestConfig} config 请求配置
 * @param {object} extra 额外配置
 * @returns {Promise<APIResponse<T>>}
 */
export function request2<T>(
  url: string | AxiosRequestConfig,
  config?: AxiosRequestConfig,
  extra?: {
    onFulfilled?: (value: AxiosResponse<any>) => APIResponse<T>;
    onRejected?: (error) => any;
  },
): Promise<APIResponse<T>> {
  const defaultFulfilled = response => response.data as APIResponse<T>;
  const defaultRejected = error => {
    throw standardErrorMessageTransformer(error);
  };
  
  const axiosPromise =
    typeof url === 'string' ? instance(url, config) : instance(url);
    
  return axiosPromise
    .then(extra?.onFulfilled || defaultFulfilled, unAuthorizationErrorHandler)
    .catch(extra?.onRejected || defaultRejected);
}

// 带响应头的请求
export function requestWithHeader(
  url: string | AxiosRequestConfig,
  config?: AxiosRequestConfig,
) {
  return request2(url, config, {
    onFulfilled: response => {
      return [response.data, response.headers] as any;
    },
  }) as any;
}
```

### 5.3 API调用示例
```typescript
// GET请求
const { data } = await request2<User>({
  url: '/users',
  method: 'GET',
});

// POST请求
const { data } = await request2<User>({
  url: '/users/login',
  method: 'POST',
  data: params,
});

// 带错误处理的请求
const { data } = await request2<User>(
  {
    url: '/users',
    method: 'GET',
  },
  undefined,
  {
    onRejected(error) {
      console.error('请求失败:', error);
      // 自定义错误处理
    },
  },
);
```

## 6. 异常处理规范

### 6.1 全局异常处理
```typescript
// 401未授权处理
function unAuthorizationErrorHandler(error) {
  if (error?.response?.status === 401) {
    message.error({ key: '401', content: i18next.t('global.401') });
    removeToken();
    return true;
  }
  throw error;
}

// 标准错误消息转换
function standardErrorMessageTransformer(error) {
  if (error?.response?.data?.message) {
    console.log('Unhandled Exception | ', error?.response?.data?.message);
    return error?.response?.data?.message;
  }
  return error;
}
```

### 6.2 组件级异常处理
```typescript
export function errorHandle(error) {
  if (error?.response) {
    // AxiosError
    const { response } = error as AxiosError;
    switch (response?.status) {
      case 401:
        message.error({ key: '401', content: i18next.t('global.401') });
        removeToken();
        break;
      default:
        message.error(response?.data.message || error.message);
        break;
    }
  } else if (error?.message) {
    // Error
    message.error(error.message);
  } else {
    message.error(error);
  }
  return error;
}

// 获取错误消息
export function getErrorMessage(error) {
  if (typeof error === 'string') {
    return error;
  }
  if (error?.response) {
    const { response } = error as AxiosError;
    switch (response?.status) {
      case 401:
        removeToken();
        return i18next.t('global.401');
      default:
        return response?.data.message || error.message;
    }
  }
  return error?.message;
}
```

### 6.3 Redux异常处理
```typescript
// Redux action错误处理
export function reduxActionErrorHandler(errorAction) {
  if (errorAction?.payload) {
    message.error(errorAction?.payload);
  } else if (errorAction?.error) {
    message.error(errorAction?.error.message);
  }
}

// Thunk错误处理
export function rejectHandle(error, rejectWithValue) {
  if (error?.response?.status === 401) {
    removeToken();
  }
  if ((error as AxiosError).response) {
    return rejectWithValue(
      ((error as AxiosError).response as AxiosResponse<APIResponse<any>>).data
        .message,
    );
  } else {
    return rejectWithValue(error.message);
  }
}
```

## 7. 样式编写规范

### 7.1 Styled Components使用
```typescript
import styled from 'styled-components/macro';
import {
  BORDER_RADIUS,
  LINE_HEIGHT_ICON_LG,
  SPACE_MD,
  SPACE_XS,
} from 'styles/StyleConstants';

// 基础样式组件
const Links = styled.div`
  display: flex;
`;

// 继承Ant Design组件样式
const LinkButton = styled(Link)`
  flex: 1;
  line-height: ${LINE_HEIGHT_ICON_LG};

  &:nth-child(2) {
    text-align: right;
  }
`;

// 使用主题变量
const UserPanel = styled.div`
  display: flex;
  padding: ${SPACE_MD};
  margin: ${SPACE_MD} 0;
  cursor: pointer;
  background-color: ${p => p.theme.bodyBackground};
  border-radius: ${BORDER_RADIUS};

  &:hover {
    background-color: ${p => p.theme.emphasisBackground};
  }

  p {
    flex: 1;
  }

  span {
    color: ${p => p.theme.textColorLight};
  }
`;
```

### 7.2 样式常量定义
```typescript
// styles/StyleConstants.ts
export const FONT_FAMILY = '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto';
export const FONT_SIZE_BODY = '14px';
export const FONT_WEIGHT_REGULAR = 400;

export const SPACE_XS = '8px';
export const SPACE_MD = '16px';
export const SPACE_LG = '24px';

export const BORDER_RADIUS = '4px';
export const LINE_HEIGHT_ICON_LG = '32px';
export const LINE_HEIGHT_ICON_XXL = '48px';
```

### 7.3 响应式设计
```typescript
const ResponsiveContainer = styled.div`
  display: flex;
  flex-direction: column;

  @media (min-width: 768px) {
    flex-direction: row;
  }

  @media (min-width: 1200px) {
    max-width: 1200px;
    margin: 0 auto;
  }
`;
```

## 8. 国际化规范

### 8.1 i18n Hook使用
```typescript
import usePrefixI18N from 'app/hooks/useI18NPrefix';

export function LoginForm() {
  // 使用前缀的国际化Hook
  const t = usePrefixI18N('login');
  const tg = usePrefixI18N('global');

  return (
    <Form>
      <Form.Item
        name="username"
        rules={[
          {
            required: true,
            message: `${t('username')}${tg('validation.required')}`,
          },
        ]}
      >
        <Input placeholder={t('username')} size="large" />
      </Form.Item>
    </Form>
  );
}
```

### 8.2 国际化资源结构
```json
// locales/zh-CN/login.json
{
  "login": "登录",
  "username": "用户名",
  "password": "密码",
  "forgotPassword": "忘记密码",
  "register": "注册",
  "alreadyLoggedIn": "您已登录",
  "enter": "进入",
  "switch": "切换用户",
  "authTitle": "第三方登录"
}

// locales/zh-CN/global.json
{
  "validation": {
    "required": "不能为空"
  },
  "401": "登录已过期，请重新登录"
}
```

### 8.3 自定义i18n Hook
```typescript
function usePrefixI18N(prefix?: string) {
  const { t, i18n } = useTranslation();
  const { i18NConfigs: vizI18NConfigs } = useContext(ChartI18NContext);

  const cachedTranslateFn = useCallback(
    (key: string, disablePrefix: boolean = false, options?: any) => {
      let translationKey = key;
      const usePrefix =
        !disablePrefix && !translationKey.includes(DATART_TRANSLATE_HOLDER);
      if (usePrefix && prefix) {
        translationKey = `${prefix}.${translationKey}`;
      }

      const langTrans = vizI18NConfigs?.find(c =>
        c.lang.includes(i18n.language),
      )?.translation;
      const contextTranslation = get(langTrans, translationKey);
      return (
        contextTranslation ||
        (t.call(Object.create(null), translationKey, options) as string)
      );
    },
    [i18n.language, prefix, t, vizI18NConfigs],
  );

  return cachedTranslateFn;
}
```

## 9. 表单处理规范

### 9.1 Ant Design Form使用
```typescript
import { Form, Input, Button } from 'antd';

export function LoginForm({ onLogin }: LoginFormProps) {
  const [form] = Form.useForm();

  return (
    <Form form={form} onFinish={onLogin}>
      <Form.Item
        name="username"
        rules={[
          {
            required: true,
            message: `${t('username')}${tg('validation.required')}`,
          },
        ]}
      >
        <Input placeholder={t('username')} size="large" />
      </Form.Item>
      
      <Form.Item
        name="password"
        rules={[
          {
            required: true,
            message: `${t('password')}${tg('validation.required')}`,
          },
        ]}
      >
        <Input placeholder={t('password')} type="password" size="large" />
      </Form.Item>
      
      <Form.Item className="last" shouldUpdate>
        {() => (
          <Button
            type="primary"
            htmlType="submit"
            size="large"
            loading={loading}
            disabled={
              loading ||
              !!form.getFieldsError().filter(({ errors }) => errors.length)
                .length
            }
            block
          >
            {t('login')}
          </Button>
        )}
      </Form.Item>
    </Form>
  );
}
```

### 9.2 表单验证规范
```typescript
// 自定义验证规则
const validatePassword = (rule: any, value: string) => {
  if (!value) {
    return Promise.reject('密码不能为空');
  }
  if (value.length < 6) {
    return Promise.reject('密码长度不能少于6位');
  }
  return Promise.resolve();
};

// 表单项配置
<Form.Item
  name="password"
  rules={[
    { required: true, message: '请输入密码' },
    { validator: validatePassword },
  ]}
>
  <Input.Password placeholder="请输入密码" />
</Form.Item>
```

## 10. 工具函数规范

### 10.1 通用工具函数
```typescript
// UUID生成
import { default as uuidv4 } from 'uuid/dist/umd/uuidv4.min';

export { uuidv4 };

// 兼容性UUID生成
export function universalUUID() {
  return typeof crypto === 'undefined'
    ? `_${Math.random().toString(36).substring(2)}`
    : uuidv4();
}

// 类名合并
export const mergeClassNames = (origin, added) =>
  classnames({ [origin]: !!origin, [added]: true });

// 阻止事件冒泡
export function stopPPG(e) {
  e.stopPropagation();
}
```

### 10.2 数据处理工具
```typescript
// 列表转树形结构
export function listToTree<
  T extends {
    id: string;
    name: string;
    parentId: string | null;
    isFolder: boolean;
    index: number | null;
  },
>(
  list: undefined | T[],
  parentId: null | string = null,
  parentPath: string[] = [],
  options?: {
    getIcon?: (o: T) => ReactElement | undefined;
    getDisabled?: (o: T, path: string[]) => boolean;
    getSelectable?: (o: T) => boolean;
    filter?: (path: string[], o: T) => boolean;
  },
): undefined | any[] {
  if (!list) {
    return list;
  }

  const treeNodes: any[] = [];
  const childrenList: T[] = [];

  list.forEach(o => {
    const path = parentPath.concat(o.id);
    if (options?.filter && !options.filter(path, o)) {
      return false;
    }
    if (o.parentId === parentId) {
      treeNodes.push({
        ...o,
        key: o.id,
        title: o.name,
        value: o.id,
        path,
        ...(options?.getIcon && { icon: options.getIcon(o) }),
        ...(options?.getDisabled && { disabled: options.getDisabled(o, path) }),
        ...(options?.getSelectable && { selectable: options.getSelectable(o) }),
      });
    } else {
      childrenList.push(o);
    }
  });

  treeNodes.sort((a, b) => Number(a.index) - Number(b.index));

  return treeNodes.map(node => {
    const children = listToTree(childrenList, node.key, node.path, options);
    return children?.length ? { ...node, children } : { ...node, isLeaf: true };
  });
}
```

### 10.3 性能优化工具
```typescript
// 文本宽度计算
let utilCanvas: null | HTMLCanvasElement = null;

export const getTextWidth = (
  text: string,
  fontWeight: string = `${FONT_WEIGHT_REGULAR}`,
  fontSize: string = FONT_SIZE_BODY,
  fontFamily: string = FONT_FAMILY,
): number => {
  const canvas = utilCanvas || (utilCanvas = document.createElement('canvas'));
  const context = canvas.getContext('2d');
  if (context) {
    context.font = `${fontWeight} ${fontSize} ${fontFamily}`;
    const metrics = context.measureText(text);
    return Math.ceil(metrics.width);
  }
  return 0;
};

// 快速删除数组元素
export function fastDeleteArrayElement(arr: any[], index: number) {
  arr[index] = arr[arr.length - 1];
  arr.pop();
}
```

## 11. 测试规范

### 11.1 测试配置
```json
// jest.config.js
{
  "testMatch": [
    "<rootDir>/src/**/__tests__/**/*.{spec,test}.{js,jsx,ts,tsx}"
  ],
  "coverageReporters": [
    "html",
    "lcov",
    "text-summary"
  ],
  "collectCoverageFrom": [
    "src/**/*.{js,jsx,ts,tsx}",
    "!src/**/*/*.d.ts",
    "!src/**/*/Loadable.{js,jsx,ts,tsx}",
    "!src/**/*/messages.ts",
    "!src/**/*/types.ts",
    "!src/index.tsx"
  ],
  "coverageThreshold": {
    "global": {
      "branches": 90,
      "functions": 90,
      "lines": 90,
      "statements": 90
    }
  }
}
```

### 11.2 组件测试示例
```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { LoginForm } from '../LoginForm';

describe('LoginForm', () => {
  const defaultProps = {
    loading: false,
    oauth2Clients: [],
    onLogin: jest.fn(),
  };

  it('should render login form correctly', () => {
    render(<LoginForm {...defaultProps} />);
    
    expect(screen.getByPlaceholderText('用户名')).toBeInTheDocument();
    expect(screen.getByPlaceholderText('密码')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '登录' })).toBeInTheDocument();
  });

  it('should call onLogin when form is submitted', () => {
    const onLogin = jest.fn();
    render(<LoginForm {...defaultProps} onLogin={onLogin} />);
    
    fireEvent.change(screen.getByPlaceholderText('用户名'), {
      target: { value: 'testuser' }
    });
    fireEvent.change(screen.getByPlaceholderText('密码'), {
      target: { value: 'password123' }
    });
    fireEvent.click(screen.getByRole('button', { name: '登录' }));
    
    expect(onLogin).toHaveBeenCalledWith({
      username: 'testuser',
      password: 'password123'
    });
  });
});
```

## 12. 代码质量规范

### 12.1 ESLint配置
```json
// .eslintrc.js
{
  "extends": [
    "react-app",
    "react-app/jest",
    "prettier"
  ],
  "plugins": [
    "prettier",
    "react-hooks",
    "jsdoc"
  ],
  "rules": {
    "prettier/prettier": "error",
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",
    "@typescript-eslint/no-unused-vars": "error",
    "prefer-const": "error",
    "no-var": "error"
  }
}
```

### 12.2 Prettier配置
```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

### 12.3 代码注释规范
```typescript
/**
 * 用户登录表单组件
 * 
 * @param props - 组件属性
 * @param props.loading - 登录加载状态
 * @param props.onLogin - 登录回调函数
 * @returns 登录表单JSX元素
 */
export const LoginForm: FC<LoginFormProps> = ({
  loading,
  onLogin,
}) => {
  // 组件实现
};

/**
 * 统一HTTP请求工具
 * 功能特性:
 *  1. 支持自定义成功和失败处理器
 *  2. 支持默认后端服务响应错误处理
 *  3. 支持Redux rejected action处理
 * 
 * @template T - 响应数据类型
 * @param {string | AxiosRequestConfig} url - 请求URL或配置
 * @param {AxiosRequestConfig} config - 请求配置
 * @param {object} extra - 额外配置
 * @returns {Promise<APIResponse<T>>} API响应Promise
 */
export function request2<T>(
  url: string | AxiosRequestConfig,
  config?: AxiosRequestConfig,
  extra?: {
    onFulfilled?: (value: AxiosResponse<any>) => APIResponse<T>;
    onRejected?: (error) => any;
  },
): Promise<APIResponse<T>> {
  // 实现代码
}
```

## 13. 性能优化规范

### 13.1 组件优化
```typescript
// 使用 React.memo 优化组件渲染
const ExpensiveComponent = React.memo(({ data, onUpdate }) => {
  // 组件实现
}, (prevProps, nextProps) => {
  // 自定义比较逻辑
  return prevProps.data.id === nextProps.data.id;
});

// 使用 useMemo 优化计算
const processedData = useMemo(() => {
  return expensiveDataProcessing(rawData);
}, [rawData]);

// 使用 useCallback 优化事件处理函数
const handleClick = useCallback((id: string) => {
  onItemClick(id);
}, [onItemClick]);
```

### 13.2 列表渲染优化
```typescript
// 使用 React.memo 优化列表项
const ListItem = React.memo(({ item, onUpdate }) => {
  return (
    <div key={item.id}>
      {item.name}
      <button onClick={() => onUpdate(item.id)}>更新</button>
    </div>
  );
});

// 虚拟滚动优化长列表
import { FixedSizeList as List } from 'react-window';

const VirtualizedList = ({ items }) => (
  <List
    height={600}
    itemCount={items.length}
    itemSize={50}
    itemData={items}
  >
    {({ index, style, data }) => (
      <div style={style}>
        <ListItem item={data[index]} />
      </div>
    )}
  </List>
);
```

### 13.3 代码分割和懒加载
```typescript
// 路由级别的代码分割
import { lazy, Suspense } from 'react';

const LoginPage = lazy(() => import('./pages/LoginPage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));

function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/dashboard" element={<DashboardPage />} />
      </Routes>
    </Suspense>
  );
}

// 组件级别的懒加载
const LazyComponent = lazy(() => 
  import('./HeavyComponent').then(module => ({
    default: module.HeavyComponent
  }))
);
```

## 14. 构建和部署规范

### 14.1 构建配置
```javascript
// craco.config.js
const path = require('path');
const MonacoWebpackPlugin = require('monaco-editor-webpack-plugin');

module.exports = {
  webpack: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
    plugins: [
      new MonacoWebpackPlugin({
        languages: ['sql', 'javascript', 'typescript'],
      }),
    ],
    configure: (webpackConfig) => {
      // 自定义webpack配置
      webpackConfig.optimization.splitChunks = {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          },
        },
      };
      return webpackConfig;
    },
  },
  devServer: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
};
```

### 14.2 环境变量配置
```bash
# .env.development
REACT_APP_API_BASE_URL=http://localhost:8080/api
REACT_APP_ENV=development
GENERATE_SOURCEMAP=true

# .env.production
REACT_APP_API_BASE_URL=/api
REACT_APP_ENV=production
GENERATE_SOURCEMAP=false
```

### 14.3 构建脚本
```json
// package.json scripts
{
  "scripts": {
    "start": "craco start",
    "build": "cross-env GENERATE_SOURCEMAP=false craco build",
    "build:analyze": "npm run build && npx source-map-explorer 'build/static/js/*.js'",
    "test": "craco test",
    "test:coverage": "npm run test -- --watchAll=false --coverage",
    "lint": "eslint --ext js,ts,tsx src",
    "lint:fix": "eslint --ext js,ts,tsx src --fix",
    "prettify": "prettier --write src"
  }
}
```

## 15. Ant Design 版本兼容性规范（重要更新）

### 15.1 Ant Design 4.16.13 版本特定规范

基于项目实际遇到的兼容性问题，以下是必须遵循的版本特定规范：

#### Modal组件属性
```typescript
// ❌ 错误：使用了v5的属性
<Modal open={visible} onCancel={handleCancel}>
  内容
</Modal>

// ✅ 正确：使用v4.16的属性
<Modal visible={visible} onCancel={handleCancel}>
  内容
</Modal>
```

#### Space组件限制
```typescript
// ❌ 错误：Space.Compact在4.16版本中不存在
<Space.Compact style={{ width: '100%' }}>
  <Select />
  <Button />
</Space.Compact>

// ✅ 正确：使用Input.Group替代
<Input.Group compact style={{ display: 'flex', width: '100%' }}>
  <Select style={{ flex: 1 }} />
  <Button />
</Input.Group>
```

#### Tag组件属性
```typescript
// ❌ 错误：size属性在4.16版本中不支持
<Tag size="small" color="blue">标签</Tag>

// ✅ 正确：移除size属性，使用CSS控制大小
<Tag color="blue" style={{ fontSize: '12px', padding: '2px 6px' }}>
  标签
</Tag>
```

### 15.2 版本兼容性检查清单

在开发新组件时，必须检查以下项目：

- [ ] Modal组件使用`visible`而非`open`属性
- [ ] 避免使用`Space.Compact`，使用`Input.Group`替代
- [ ] Tag组件不使用`size`属性
- [ ] Form组件使用4.16版本的API
- [ ] Table组件分页回调函数参数正确
- [ ] DatePicker组件使用moment而非dayjs

## 16. TypeScript错误处理最佳实践（重要更新）

### 16.1 枚举类型使用规范

基于实际开发中遇到的问题，强制要求：

```typescript
// ❌ 错误：使用字符串字面量
const status = "PUBLISHED";
if (metric.status === "PUBLISHED") {
  // 处理逻辑
}

// ✅ 正确：使用枚举类型
export enum MetricStatus {
  DRAFT = 'DRAFT',
  PUBLISHED = 'PUBLISHED',
  ARCHIVED = 'ARCHIVED',
}

const status = MetricStatus.PUBLISHED;
if (metric.status === MetricStatus.PUBLISHED) {
  // 处理逻辑
}
```

### 16.2 API响应数据安全访问

```typescript
// ❌ 错误：直接访问可能为undefined的属性
const handleResponse = (response) => {
  setData(response.data.list);
  setTotal(response.data.total);
};

// ✅ 正确：使用可选链和默认值
const handleResponse = (response) => {
  setData(response.data?.list || []);
  setTotal(response.data?.total || 0);
};
```

### 16.3 异常处理类型注解

```typescript
// ❌ 错误：catch块中error类型不明确
try {
  await submitForm(values);
} catch (error) {
  if (error.errorFields) {
    // TypeScript错误：类型"{}"上不存在属性"errorFields"
  }
}

// ✅ 正确：为error添加类型注解
try {
  await submitForm(values);
} catch (error: any) {
  if (error.errorFields) {
    // 正确处理表单验证错误
    form.setFields(error.errorFields);
  }
}
```

## 17. 组件集成规范（新增）

### 17.1 服务类组织规范

基于指标管理功能的实践，推荐使用类的方式组织API服务：

```typescript
/**
 * 指标定义服务类
 * 提供指标管理相关的API调用方法
 */
class MetricDefinitionService {
  /**
   * 获取指标定义列表
   * @param params 查询参数
   * @returns Promise<APIResponse<PageResponse<MetricDefinition>>>
   */
  async getMetricDefinitions(
    params?: MetricDefinitionQueryParam,
  ): Promise<APIResponse<PageResponse<MetricDefinition>>> {
    return request2<PageResponse<MetricDefinition>>({
      url: '/api/v1/metrics',
      method: 'GET',
      params,
    });
  }

  /**
   * 创建指标定义
   * @param data 创建参数
   * @returns Promise<APIResponse<MetricDefinition>>
   */
  async createMetricDefinition(
    data: MetricDefinitionCreateParam,
  ): Promise<APIResponse<MetricDefinition>> {
    return request2<MetricDefinition>({
      url: '/api/v1/metrics',
      method: 'POST',
      data,
    });
  }
}

// 导出单例实例
export const metricDefinitionService = new MetricDefinitionService();
```

### 17.2 页面组件结构规范

```typescript
/**
 * 指标管理页面组件
 * 
 * 功能特性：
 * - 指标列表展示和搜索
 * - 指标创建、编辑、删除
 * - 指标状态管理
 */
export const MetricManagementPage: FC<MetricManagementPageProps> = () => {
  const t = usePrefixI18N('metric');
  const tg = usePrefixI18N('global');

  // 状态定义
  const [loading, setLoading] = useState(false);
  const [metrics, setMetrics] = useState<MetricDefinition[]>([]);
  const [total, setTotal] = useState(0);
  const [queryParams, setQueryParams] = useState<MetricDefinitionQueryParam>({
    page: 1,
    size: 20,
  });

  // 数据加载
  const loadMetrics = useCallback(async () => {
    try {
      setLoading(true);
      const response = await metricDefinitionService.getMetricDefinitions(queryParams);
      setMetrics(response.data?.list || []);
      setTotal(response.data?.total || 0);
    } catch (error) {
      message.error(getErrorMessage(error));
    } finally {
      setLoading(false);
    }
  }, [queryParams]);

  // 使用useCallback优化事件处理函数
  const handleSearch = useCallback(
    (keyword: string) => {
      setQueryParams(prev => ({
        ...prev,
        name: keyword || undefined,
        page: 1,
      }));
    },
    [],
  );

  // 使用useMemo优化计算属性
  const columns: ColumnsType<MetricDefinition> = useMemo(
    () => [
      {
        title: t('definition.name'),
        dataIndex: 'name',
        key: 'name',
        ellipsis: true,
      },
      // 其他列定义...
    ],
    [t, handleDetail, handleEdit],
  );

  // 副作用处理
  useEffect(() => {
    loadMetrics();
  }, [loadMetrics]);

  return (
    <StyledContainer>
      {/* 组件JSX */}
    </StyledContainer>
  );
};
```

## 18. 路由和菜单集成规范（新增）

### 18.1 路由配置规范

```typescript
// 在MainPage/index.tsx中添加路由
const MetricPage = lazy(() => import('./pages/MetricPage'));

// 路由配置
<Route path="/metrics" element={<MetricPage />} />
```

### 18.2 菜单配置规范

```typescript
// 在Navbar/index.tsx中添加菜单项
const navs = [
  // 其他菜单项...
  {
    key: 'metrics',
    title: t('nav.metrics'),
    icon: <DatabaseOutlined />,
    path: '/metrics',
    // 权限控制
    permission: 'metric:view',
  },
];
```

### 18.3 国际化资源更新

```json
// locales/zh/translation.json
{
  "main": {
    "nav": {
      "metrics": "指标管理"
    }
  },
  "metric": {
    "title": "指标管理",
    "definition": {
      "name": "指标名称",
      "description": "指标描述"
    }
  }
}
```

## 19. 最佳实践总结（更新）

### 19.1 代码组织原则
- **单一职责**: 每个组件和函数只负责一个功能
- **关注点分离**: 业务逻辑、UI逻辑、状态管理分离
- **可复用性**: 提取通用组件和工具函数
- **可测试性**: 编写易于测试的纯函数和组件
- **版本兼容**: 严格遵循Ant Design版本限制

### 19.2 性能优化原则
- **避免不必要的渲染**: 使用React.memo、useMemo、useCallback
- **代码分割**: 按路由和功能模块进行代码分割
- **资源优化**: 图片压缩、字体优化、CDN使用
- **缓存策略**: 合理使用浏览器缓存和应用缓存

### 19.3 用户体验原则
- **加载状态**: 提供明确的加载反馈
- **错误处理**: 友好的错误提示和降级方案
- **响应式设计**: 适配不同设备和屏幕尺寸
- **无障碍访问**: 支持键盘导航和屏幕阅读器

### 19.4 开发效率原则
- **类型安全**: 充分利用TypeScript的类型检查
- **代码规范**: 统一的代码风格和命名规范
- **自动化**: 使用工具自动化代码格式化、测试、构建
- **文档完善**: 编写清晰的代码注释和组件文档

## 20. 常见问题和解决方案（重要更新）

### 20.1 TypeScript编译错误

#### 问题1：枚举类型使用错误
```typescript
// 问题：不能将类型""PUBLISHED""分配给类型"MetricStatus | undefined"
// 解决：使用枚举而非字符串字面量
import { MetricStatus } from 'app/types/MetricDefinition';
const status = MetricStatus.PUBLISHED;
```

#### 问题2：API响应数据结构错误
```typescript
// 问题：类型"PageResponse<T>"的参数不能赋给类型"SetStateAction<T[]>"
// 解决：正确解构分页响应数据
const response = await api.getList();
setData(response.data?.list || []);
setTotal(response.data?.total || 0);
```

### 20.2 Ant Design版本兼容问题

#### 问题1：Modal组件属性错误
```typescript
// 问题：类型"IntrinsicAttributes & ModalProps"上不存在属性"open"
// 解决：使用visible属性
<Modal visible={visible} onCancel={handleCancel} />
```

#### 问题2：Space.Compact组件不存在
```typescript
// 问题：Space.Compact在4.16版本中不存在
// 解决：使用Input.Group替代
<Input.Group compact>
  <Select style={{ flex: 1 }} />
  <Button />
</Input.Group>
```

### 20.3 状态管理问题
```typescript
// 问题：状态更新不及时
// 解决：使用useEffect监听状态变化
useEffect(() => {
  if (user) {
    updateUserProfile(user);
  }
}, [user]);

// 问题：状态更新导致无限循环
// 解决：正确设置依赖数组
const fetchData = useCallback(async () => {
  const data = await api.getData();
  setData(data);
}, []); // 空依赖数组，避免无限循环
```

### 20.4 性能问题
```typescript
// 问题：组件频繁重渲染
// 解决：使用React.memo和依赖优化
const ExpensiveComponent = React.memo(({ data }) => {
  const processedData = useMemo(() => {
    return heavyComputation(data);
  }, [data]);

  return <div>{processedData}</div>;
});

// 问题：事件处理函数重复创建
// 解决：使用useCallback缓存函数
const handleClick = useCallback((id: string) => {
  onItemClick(id);
}, [onItemClick]);
```

## 21. 编码规范检查清单（新增）

### 21.1 TypeScript规范检查
- [ ] 接口定义使用PascalCase命名
- [ ] 使用枚举替代字符串联合类型
- [ ] API响应使用泛型类型定义
- [ ] 组件Props定义完整的类型
- [ ] 使用可选链安全访问对象属性

### 21.2 React组件规范检查
- [ ] 使用FC类型定义函数组件
- [ ] 事件处理函数使用useCallback包装
- [ ] 计算属性使用useMemo优化
- [ ] 正确设置useEffect依赖数组
- [ ] 使用React.memo优化组件渲染

### 21.3 Ant Design兼容性检查
- [ ] Modal组件使用visible而非open属性
- [ ] 避免使用Space.Compact组件
- [ ] Tag组件不使用size属性
- [ ] 表单验证使用4.16版本API
- [ ] 分页组件回调函数参数正确

### 21.4 API请求规范检查
- [ ] 统一使用request2方法
- [ ] 正确处理API响应数据结构
- [ ] 实现完整的错误处理机制
- [ ] 使用类组织API服务方法
- [ ] 添加完整的JSDoc注释

### 21.5 国际化规范检查
- [ ] 使用usePrefixI18N Hook
- [ ] 国际化资源结构化组织
- [ ] 提供中英文双语支持
- [ ] 错误消息使用国际化键
- [ ] 表单验证消息国际化

## 22. 总结

本编码规范基于Datart项目的实际开发经验，特别是指标管理功能的开发过程中遇到的问题和解决方案进行了全面更新。主要改进包括：

### 22.1 新增内容
1. **Ant Design版本兼容性规范** - 解决实际开发中的版本兼容问题
2. **TypeScript错误处理最佳实践** - 基于真实错误案例的解决方案
3. **组件集成规范** - 标准化的组件开发和集成流程
4. **路由和菜单集成规范** - 完整的功能模块集成指南
5. **编码规范检查清单** - 可操作的代码质量检查标准

### 22.2 重点强调
1. **类型安全** - 强制使用枚举类型，避免字符串字面量
2. **版本兼容** - 严格遵循Ant Design 4.16.13版本限制
3. **错误处理** - 完善的异常处理和用户反馈机制
4. **性能优化** - 合理使用React性能优化Hooks
5. **代码质量** - 统一的代码风格和注释规范

### 22.3 实践价值
通过遵循本规范，可以确保：
- **零TypeScript编译错误**
- **完整的版本兼容性**
- **高质量的用户体验**
- **可维护的代码结构**
- **团队协作的一致性**

本规范将持续更新，以适应项目发展和技术演进的需要。
