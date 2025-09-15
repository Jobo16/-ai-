import csv
import asyncio
import aiohttp
import json
from typing import List, Tuple
import time

class SiliconCloudTranslator:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.siliconflow.cn/v1/chat/completions"
        self.headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {api_key}"
        }
        
    async def translate_text(self, session: aiohttp.ClientSession, text: str) -> str:
        "翻译单个文本"
        prompt = f"""
Dưới đây là thông tin câu hỏi:

{text}"
"""
        
        payload = {
            "model": "deepseek-ai/DeepSeek-V3.1",
            "thinking_budget": 4096,
            "top_p": 0.7,
            "stream": False,
            "max_tokens": 4096,
            "temperature": 1,
            "messages": [
                {"role": "system", "content": 
"""
Bạn là một giáo viên tiếng Trung chuyên nghiệp, có chuyên môn phân tích đáp án đề thi. Dựa trên thông tin câu hỏi được nhập, hãy trả lời phần giải thích đáp án bằng tiếng Việt theo hai phần:
1.Giải thích lý do lựa chọn đúng 
2.Phân tích các phương án gây nhiễu còn lại.


Trả lời ngắn gọn, không thêm nội dung không liên quan(trước khi phân tích cần giải thích ngắn gọn ý nghĩa của các phương án bằng tiếng Việt).
"""
},
                {"role": "user", "content": prompt}
            ]
        }
        
        try:
            async with session.post(self.base_url, headers=self.headers, json=payload) as response:
                if response.status == 200:
                    result = await response.json()
                    return result['choices'][0]['message']['content'].strip()
                else:
                    # 获取详细错误信息
                    error_text = await response.text()
                    print(f"API请求失败，状态码: {response.status}")
                    print(f"错误详情: {error_text}")
                    print(f"请求文本长度: {len(text)} 字符")
                    print(f"请求文本前100字符: {text[:100]}...")
                    return f"翻译失败: {response.status} - {error_text[:200]}"
        except Exception as e:
            print(f"翻译出错: {str(e)}")
            return f"翻译出错: {str(e)}"
    
    async def translate_batch(self, texts: List[Tuple[int, str]]) -> List[Tuple[int, str, str]]:
        "批量翻译"
        connector = aiohttp.TCPConnector(limit=100, limit_per_host=50)
        timeout = aiohttp.ClientTimeout(total=300)
        
        async with aiohttp.ClientSession(connector=connector, timeout=timeout) as session:
            # 创建所有翻译任务
            tasks = []
            for idx, text in texts:
                task = asyncio.create_task(self.translate_text(session, text))
                tasks.append((idx, text, task))
            
            # 并发执行所有任务
            results = []
            completed_tasks = await asyncio.gather(*[task for _, _, task in tasks], return_exceptions=True)
            
            for i, (idx, original_text, _) in enumerate(tasks):
                translated = completed_tasks[i]
                if isinstance(translated, Exception):
                    translated = f"翻译出错: {str(translated)}"
                results.append((idx, original_text, translated))
                print(f"完成翻译 {idx + 1}: {original_text[:50]}...")
            
            return results

def read_csv_data(file_path: str) -> List[Tuple[int, str]]:
    "读取CSV文件中的中文数据"
    data = []
    with open(file_path, 'r', encoding='utf-8') as file:
        reader = csv.reader(file)
        next(reader)  # 跳过标题行
        for idx, row in enumerate(reader):
            if row and len(row) > 0 and row[0].strip():  # 确保有中文内容
                data.append((idx, row[0].strip()))
    return data

def save_translations(file_path: str, translations: List[Tuple[int, str, str]], start_idx: int = 0):
    "保存翻译结果到CSV文件"
    # 读取现有数据
    existing_data = []
    try:
        with open(file_path, 'r', encoding='utf-8') as file:
            reader = csv.reader(file)
            existing_data = list(reader)
    except FileNotFoundError:
        existing_data = [['中文', '英语']]  # 创建标题行
    
    # 更新翻译结果
    for idx, original, translated in translations:
        row_idx = idx + 1  # 考虑标题行
        if row_idx < len(existing_data):
            if len(existing_data[row_idx]) < 2:
                existing_data[row_idx].append(translated)
            else:
                existing_data[row_idx][1] = translated
        else:
            # 如果行不存在，添加新行
            existing_data.append([original, translated])
    
    # 写回文件
    with open(file_path, 'w', encoding='utf-8', newline='') as file:
        writer = csv.writer(file)
        writer.writerows(existing_data)
    
    print(f"已保存 {len(translations)} 条翻译结果")

async def main():
    # 配置
    api_key = "your-api"
    input_file = "翻译表.csv"
    batch_size = 20  # 每批处理50个，增加并发数
    
    # 初始化翻译器
    translator = SiliconCloudTranslator(api_key)
    
    # 读取数据
    print("正在读取CSV文件...")
    data = read_csv_data(input_file)
    print(f"共找到 {len(data)} 条待翻译数据")
    
    # 分批处理
    total_batches = (len(data) + batch_size - 1) // batch_size
    
    for batch_num in range(total_batches):
        start_idx = batch_num * batch_size
        end_idx = min(start_idx + batch_size, len(data))
        batch_data = data[start_idx:end_idx]
        
        print(f"\n正在处理第 {batch_num + 1}/{total_batches} 批 (第 {start_idx + 1}-{end_idx} 条)...")
        
        # 翻译当前批次
        translations = await translator.translate_batch(batch_data)
        
        # 保存结果
        save_translations(input_file, translations, start_idx)
        
        print(f"第 {batch_num + 1} 批翻译完成并已保存")
        
        # 减少延迟时间
        if batch_num < total_batches - 1:
            print("等待0.5秒后继续下一批...")
            await asyncio.sleep(0.5)
    
    print("\n所有翻译任务完成！")

if __name__ == "__main__":
    asyncio.run(main())
