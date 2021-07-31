# 前言
在产品版本迭代的过程中，测试团队大部分是依据产品设计文档或者需求文档进行用例设计，无论是通过自动化测试的手段还是手工测试的手段去对产品的缺陷进行挖掘，存在以下问题：
1.  不易得知用例有没有覆盖到该次代码的所有改动点。
2.  虽然有jacoco现成的全量代码的覆盖率，但是针对于本次的代码改动不直观，不能确认本次的改动用例覆盖了多少，
增量代码覆盖率的出现解决了这个痛点，赋能团队精准测试的能力。

所谓增量代码覆盖率，即比较两个分支(一般是测试分支和master分支)的代码的diff，将全量代码覆盖率的范围缩小到本次diff涉及到的所有函数方法的颗粒度，非本次code diff设计到的函数方法均被在统计中剔除。
# 方案设计
![](https://assets.che300.com/wiki/2021-07-28/16274595821893212.png)
由于jacoco core模块的代码进行了改造，jacoco-cli.jar需要重新打包
1. 基于git 的diff功能计算出两个不同分支的差异代码行信息
2. 通过jdt将diff行数转换为diff函数方法级别
3. 在jacoco core模块根据 diff函数方法信息 过滤 统计的函数方法范围
# 差异代码细节
1. 获取差异代码细节
     ```java
    public Map<String, List<MethodInfo>> run_diff(){
        //通过gitlab接口提取两个分支的差异行数据
        JSONArray diff_src = git.get_commit_diff(from_commit,to_commit);
        //序列化成java对象
        List<DiffInfo> diff = DiffParse.parse(diff_src);
        //过去没有变化的文件
        filter(diff);
        //将diff行信息转换为diff差异信息
        parse_diff_method(diff);
        return diff_class_exist_method;
    }
    
    
    /*
    *通过gitlab的api获取两个分支的diff详情
    */
    public JSONArray get_commit_diff(String from, String to){
        String uri = "/api/v4/projects/" + this.project_id + "/repository/compare";
        Map<String, String> params = new HashMap<>();
        params.put("from",from);
        params.put("to",to);
        JSONObject res = get(uri,params,this.headers);
        return (JSONArray) res.get("diffs");
    }
   
   
   
    /*
     *序列化数据
     */
   public static List<DiffInfo> parse(JSONArray data_array){
        List<DiffInfo> diffInfos = new ArrayList<>();
        for (Object data:data_array){
            JSONObject data1 = (JSONObject) data;
            DiffInfo diffInfo = new DiffInfo();
            //设置diff文件地址
            diffInfo.setFile_path(data1.get("new_path").toString());
            // 设置diff类型
            diffInfo.setType(parseDiffType(data1));
            // 解析diff 行数
           diffInfo.setLines(parseDiffContent(data1.get("diff").toString()));
           diffInfos.add(diffInfo);
        }
        return diffInfos;
    }
    
    
     /**
     * 过滤类型为 空白的、 删除的 diff
     * 过滤不是java结尾的diff
     * @param data
     */
    private void filter(List<DiffInfo> data){
        data.removeIf(file -> file.getType() == DiffType.DELETED || file.getType() == DiffType.EMPTY);
        data.removeIf(file->!file.getFile_path().endsWith(".java"));
    }
    
    
     /**
     * 将diff出的行数转换成方法
     * @param data
     */
    private void parse_diff_method(List<DiffInfo> data){
        data.forEach(item ->{
            String file_path = item.getFile_path();
            String java_text = git.get_file_content(file_path, to_commit);
            //ast解析java源码
            ASTParse ast_parse = new ASTParse(file_path,java_text);
            List<Integer> diff_lines = item.getLines();
            //解析出行对应的类
            List<ClassInfo> diff_tree = ast_parse.parse_code_class();
            lines_to_method(diff_tree,diff_lines);
        });
    ```

2.  jacoco改动源码细节
      ```java
      // org/jacoco/cli/internal/commands/Report.java
    	/**
	 * 检查是否为增量代码覆盖
	 * @return
	 */
	private boolean isDiff(PrintWriter out){
		List<String> stringList = new ArrayList<>(Arrays.asList(gitlabHost,gitlabToken,gitlabProjectId,fromCommit,toCommit));
		return stringList.stream().noneMatch(StringUtils::isEmptyOrNull);
	}
	private IBundleCoverage analyze(final ExecutionDataStore data,
			final PrintWriter out) throws IOException {
		final CoverageBuilder builder;
		//判断是否开启增量代码覆盖
        if (isDiff(out)){
			builder = new CoverageBuilder(gitlabHost,gitlabProjectId,gitlabToken,fromCommit,toCommit);
			out.println("[!!!INFO] === start deal with Incremental code coverage ===");
		}else{
			builder = new CoverageBuilder();
		}
		final Analyzer analyzer = new Analyzer(data, builder);
		for (final File f : classfiles) {
			analyzer.analyzeAll(f);
		}
		printNoMatchWarning(builder.getNoMatchClasses(), out);
		return builder.getBundle(name);
	}


    // org/jacoco/core/analysis/CoverageBuilder.java
    /**
	 * 增量代码 new builder
     * 接受传入的差异信息的相关入参数据
	 * **/
	public CoverageBuilder(String host, String project_id, String token, String from_commit, String to_commit) {
		this.classes = new HashMap<String, IClassCoverage>();
		this.sourcefiles = new HashMap<String, ISourceFileCoverage>();
		if (classInfos == null || classInfos.isEmpty()){
			DiffMain diffMain = new DiffMain(host, project_id, token, from_commit, to_commit);
			classInfos = diffMain.run_diff();
		}
	}
    

    // org/jacoco/core/internal/flow/ClassProbesAdapter.java
            public final MethodVisitor visitMethod(final int access, final String name,
                final String desc, final String signature,
                final String[] exceptions) {
            final MethodProbesVisitor methodProbes;
            final MethodProbesVisitor mv = cv.visitMethod(access, name, desc,
                    signature, exceptions);
    //		if (mv == null) {
    //			// We need to visit the method in any case, otherwise probe ids
    //			// are not reproducible
    //			methodProbes = EMPTY_METHOD_PROBES_VISITOR;
    //		} else {
    //			methodProbes = mv;
    //		}
            //	增量计算覆盖率
            if (mv !=null && DiffMain.is_contain_method(this.name,name,desc,CoverageBuilder.classInfos) ) {
                methodProbes = mv;
            } else {
                // We need to visit the method in any case, otherwise probe ids
                // are not reproducible
                methodProbes = EMPTY_METHOD_PROBES_VISITOR;
            }
            return new MethodSanitizer(null, access, name, desc, signature,
                    exceptions) {

                @Override
                public void visitEnd() {
                    super.visitEnd();
                    LabelFlowAnalyzer.markLabels(this);
                    final MethodProbesAdapter probesAdapter = new MethodProbesAdapter(
                            methodProbes, ClassProbesAdapter.this);
                    if (trackFrames) {
                        final AnalyzerAdapter analyzer = new AnalyzerAdapter(
                                ClassProbesAdapter.this.name, access, name, desc,
                                probesAdapter);
                        probesAdapter.setAnalyzer(analyzer);
                        methodProbes.accept(this, analyzer);
                    } else {
                        methodProbes.accept(this, probesAdapter);
                    }
                }
            };
        }
        
        
        // org/jacoco/core/internal/diff2/DiffMain.java 新增自定义方法
           public static Boolean is_contain_method(String location, String current_method,String current_method_args,Map<String, List<MethodInfo>> diffs){
            if (diffs == null){
                //如果diffs为null走全量覆盖率
                return true;
            }
            if (diffs.containsKey(location)){
                List<MethodInfo> methods = diffs.get(location);
                for (MethodInfo method:methods){
                    // 判断方法是否在diff 类中 选择方法
                    if (current_method.equals(method.getMethodName())){
                        return checkArgs(current_method_args,method.getArgs());
                    }
                }
            }
        return false;
        }

        /**
         * 判断参数是否相同，主要通过参数类型以及个数判断
         * 暂未对返回类型做校验判断，后期可优化
         * @param current_method_args_src
         * @param reference_args
         * @return
         */
        private static Boolean checkArgs(String current_method_args_src,List<ArgInfo> reference_args){
            Type[] current_method_args = Type.getArgumentTypes(current_method_args_src);
            //判断参数个数是否为空
            if (current_method_args.length ==0 && reference_args.size() ==0){
                return true;
            }
            if (current_method_args.length == reference_args.size()){
                //判断参数类型是否相同
                List<Boolean> is_same_list = new ArrayList<>();
                for (int i=0;i<current_method_args.length;i++){
                    Type current_method_arg = current_method_args[i];
                    String current_method_arg1 = current_method_arg.toString();
                    String current_method_arg_final;
                    String reference_arg_type = reference_args.get(i).getType();
                    String reference_arg_type_final = reference_arg_type;
                    // Ljava/lang/String;  / 分割 取出类名最后一个
                    //替换jvm类型为正常类型
                    if (type_map().containsKey(current_method_arg1)){
                        //如果 参数类型为jvm短标识
                        current_method_arg_final = type_map().get(current_method_arg1);
                    }else {
                        //标记参数是不是数组
                        boolean is_array = current_method_arg1.contains("[");

                        String[] current_method_arg2 = current_method_arg1.split("/");
                        String current_method_arg3 = current_method_arg2[current_method_arg2.length - 1];
    //                    String current_method_arg4 = current_method_arg3.replace(";","");
                        Pattern pattern = Pattern.compile("<.+>|;"); //去掉空格符合换行符
                        Matcher matcher = pattern.matcher(current_method_arg3);
                        String current_method_arg4 = matcher.replaceAll("");
                        reference_arg_type_final = pattern.matcher(reference_arg_type).replaceAll("");
                        // 暂不考虑二维数组
                        if (is_array) {
                            current_method_arg_final = current_method_arg4 + "[]";
                        }else {
                            current_method_arg_final = current_method_arg4;
                        }
                    }
                    is_same_list.add(current_method_arg_final.equals(reference_arg_type_final));
                }
                return is_same_list.stream().allMatch(f-> f);
            }
            return false;
        }
    ```

# 不足
1. 基于gitlab获取的git diff信息 如果diff内容过大超出gitlab的diff limit限制，diff内容为空。计划用jgit本地操作git仓库获取diff信息
# 总结
1. 增量代码覆盖率可以作为开发自测的准入标准，以确认达到提测标准
2. 测试人员可以根据增量代码覆盖率去完善自己的测试用例，进一步提升对质量的把控
3. 搭配全量代码覆盖率作为回归测试的参考，以便完善回归用例覆盖主流程逻辑。

