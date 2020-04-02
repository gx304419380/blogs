![Snipaste_2018-09-20_09-10-20.jpg](https://upload-images.jianshu.io/upload_images/13277366-63074ef71b54471f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


对齐前后的效果：

```
//对齐前
Set<Integer> idSet = organizationList.stream()
                .map(BaseOrganization::getPath)
                .flatMap(path -> Arrays.stream(path.split(ORG_NODE_PATH_SPLIT)))
                .filter(ObjectUtils::isNotEmpty)
                .map(Integer::valueOf)
                .collect(Collectors.toSet());

//对齐后
Set<Integer> idSet = organizationList.stream()
                                     .map(BaseOrganization::getPath)
                                     .flatMap(path -> Arrays.stream(path.split(ORG_NODE_PATH_SPLIT)))
                                     .filter(ObjectUtils::isNotEmpty)
                                     .map(Integer::valueOf)
                                     .collect(Collectors.toSet());
```
