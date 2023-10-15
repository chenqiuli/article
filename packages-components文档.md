### FileUpload

| 参数     | 说明                                                                                                                              | 默认值 | 类型                                                       | 是否必传 |
| -------- | --------------------------------------------------------------------------------------------------------------------------------- | ------ | ---------------------------------------------------------- | -------- |
| readOnly | 是否开启只读模式                                                                                                                  | false  | boolean                                                    | 否       |
| params   | 获取文件类型，文件列表已有数据，一键下载接口等接口的入参，dataKey - 业务单据主键，appCode - 应用编码，groupCodes - 附件类型组编码 | -      | { dataKey: string; appCode: string; groupCodes: string[] } | 是       |
| ref      | 外部创建的 ref 实例用于获取组件的方法                                                                                             | -      | any                                                        | 是       |

### fileUploadRefInstance 实例

| 名称          | 说明         | 类型                                                | 默认值 |
| ------------- | ------------ | --------------------------------------------------- | ------ |
| validateFiles | 触发必填校验 | { customize: boolean } => `Promise<ValidateReturn>` | { }    |

### ValidateReturn

| 参数   | 说明                 | 类型           |
| ------ | -------------------- | -------------- |
| values | 文件列表数据         | ReturnValues[] |
| errors | 缺失文件的文件类型   | ReturnErrors[] |
| status | 是否有文件正在上传中 | boolean        |

### validateFiles 返回示例

```tsx
validateFiles({}).then((res: ValidateReturn) => {
  const { values, errors, status } = res;
  /*
  values:
    {
      fileId: "3a0e1ad5-e05e-0ccc-1560-e6a69092c3ec",
      appCode: "LeaseContract",
      attachmentTypeGroup: "SignContract",
      attachmentType: "ContractText",
      dataKey: "33333333-3333-3333-3333-333333333333",
      fileRecordOperation: "Unchange",
      displayOrder: 1
    }
  */
  /*
  errors:
    {
      typeCode: "ShopDeliveryStandards",
      typeName: "商铺交付标准",
      businessObject: "Contract",
      appCode: "SignContract",
      groupCode: "LeaseContract"
    }
  */
  /*
  status: false
  */
});
```

### 代码演示

#### 采用默认校验

```tsx
import React from 'react';
import {
  FileUpload,
  ValidateReturn,
  ReturnValues,
  ReturnErrors,
} from '@ifca/components';
export default function App() {
  const fileUploadRef = React.createRef();
  return (
    <React.Fragement>
      <FileUpload
        // readOnly
        params={{
          dataKey,
          appCode,
          groupCodes,
        }}
        ref={fileUploadRef}
      />
      <Button
        type="primary"
        onClick={async () => {
          const fileUploadRefInstance: any = fileUploadRef.current;=
          fileUploadRefInstance.validateFiles({}).then((res: ValidateReturn) => {
            const { values, errors, status } = res;
            console.log(errors, 'errors')
            if (!errors?.length && !status) {
              console.log(values, '文件列表');
            }
          });
        }}
      >
        保存
      </Button>
    </React.Fragement>
  );
}
```

#### 自定义校验 validateFiles

```tsx
fileUploadRefInstance
  .validateFiles({ customize: true })
  .then((res: ValidateReturn) => {
    const { values, errors, status } = res;
    console.log(values, errors, status, 'res');
    if (errors?.length) {
      console.log('自定义必传未通过逻辑');
      return;
    } else if (status) {
      console.log('文件正在上传中逻辑');
      return;
    } else {
      console.log(values, '文件列表');
    }
  });
```
