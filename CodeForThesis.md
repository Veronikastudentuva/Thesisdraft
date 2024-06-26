# Code for Thesis 

# This code is divided into three sections. In section 1, a combination of ocr and regular expressions are used. In section 2 the method of template based zonal ocr is used. The third section details the "drag and drop" zonal ocr method. 

# Section 1: Combination of OCR and Regular Expressions

#1.1 Install and setup the required packages

#We will use the python library PyMuPDF 

    !pip install pymupdf
    !pip install PyMuPDF
    !pip install PyMuPDF googletrans==4.0.0-rc1
    !pip install googletrans==4.0.0-rc1
    !pip install deep-translator
    !pip install langdetect

    from google.colab import files
    import fitz
    from typing import TextIO
    import re
    import pandas as pd

    from langdetect import detect, DetectorFactory
    from langdetect.lang_detect_exception import LangDetectException
    from deep_translator import GoogleTranslator

#1.2 Upload the PDF document and then the text from it will be extracted with line separators to read it more clearly

    def pdfcontent(pdf_path):
        uploaded_document = fitz.open(pdf_path)
        text = ""
        for page_num in range(len(uploaded_document)):
            page = uploaded_document[page_num]
            text += page.get_text("text") + "\n"
        return text
    uploaded = files.upload()
    for pdf_file in uploaded.keys():
        text = pdfcontent(pdf_file)


#1.3 Next we translate the text to English

    DetectorFactory.seed = 0

    def clean_text(text):
        lines = text.splitlines()
        cleaned_lines = [line.strip() for line in lines if line.strip()]
        return cleaned_lines

    def detect_language(line):
      try:
        detected = detect(line)
        return detected
      except LangDetectException as e:
        return None

    def toenglish(line, source_language):
        translated = GoogleTranslator(source=source_language, target='en').translate(line)
        return translated

    for pdf_file in uploaded.keys():
        text = extract_text_from_pdf(pdf_file)
        if text:
            cleaned_lines = clean_text(text)
            translatedsingle = []
            for i, line in enumerate(cleaned_lines[:150]):
                detected_language = detect_language(line)
                if detected_language:
                    translated_line = toenglish(line, detected_language)
                    translatedsingle.append(translated_line)
                else:
                    translatedsingle.append(line)
            translated = "\n".join(translatedsingle)
            text = translated
   
#1.4 The code below is functions which use regular expressions to match patterns to look for the desired attributes:

#This function finds the key value pairs and is meant to work with different spacing, whether the value is on the same line or on a different line than the key
#As in the documents we see ":" before any attribute we use that to guide whether the value is 
#It assumes that the key is followed by ":" then the value. We account for possible differences in spacing, such as some values being within the same line as the key, while others are a line or two below the key 
#This occurs as in the pdfs the data is othen in tables on in different lines, which is then reflected as we extract the text 
#Sometimes there is only 1 key and other times there are multiple so we want to iterate through the list of keys
    
    def keyvaluepair(text, keys):
        if isinstance(keys, str):
            keys = [keys]
        lines = text.split('\n')
            value = None
        for key in keys:
            for i, line in enumerate(lines):
                if key in line:
                    match = re.search(rf'{key}\s*:\s*(.*)', line)
                    if match and match.group(1).strip():
                        value = match.group(1).strip()
                    else:
                        if i + 1 < len(lines) and lines[i + 1].strip() == '':
                            if i + 2 < len(lines) and lines[i + 2].strip() == ':':
                                if i + 3 < len(lines):
                                    value = lines[i + 3].strip()
                        elif i + 1 < len(lines) and lines[i + 1].strip() == ':':
                            if i + 2 < len(lines):
                                value = lines[i + 2].strip()
                        elif i + 1 < len(lines):
                            value = lines[i + 1].strip()
                    if value:
                    break
            if value:
                break
        return value
    
#The CAS Number always follows the same pattern     
#It can be made up from up to 10 numbers. They are separated into 3 different groups with "-". The first part has 2-7 digits, second has 2 digits and third has 1.

    def casnumber(text):
        cas_pattern = r'\b\d{2,7}-\d{2}-\d\b'
        match = re.search(cas_pattern, text)
        if match:
            return match.group(0)
        return None

    def ecnumber(text):
        ec_no_pattern = r'\b(2|4|5)\d{2}-\d{3}-\d\b'
        match = re.search(ec_no_pattern, text)
        if match:
            return match.group(0)
        return None
        
#This function relies on the previously defined function to match key value pairs
#In this case the keys are 'Identified uses', 'Substance/Mixture' and 'Recommended use' as we are assuming all those have the same meaning for our purpose

    def uses(text):
        #We want to be able to seach for these patterns both if they are in capital and lower cases so use the package "IGNORECASE" from regular expressions for a case-insesitive search
        text = re.sub(r'Recommended use', '', text, count=1, flags=re.IGNORECASE)
        if "Identified uses:" in text:
            return keyvaluepair(text, ["Identified uses"])
        elif "Substance/Mixture" in text:
            return keyvaluepair(text, ["Substance/Mixture"])
        elif "Recommended use" in text:
            return keyvaluepair(text, ["Recommended use"])
        else:
            return None

#We check if its hazardous by looking for specific phrases which say that it is not hazardous, so if such a phrase is not found, we assume it is hazardous 

    def isitahazard(text):
        lower_text = text.lower()
        if "has not been classified as dangerous" in lower_text or "not a hazardous" in lower_text or "not classified as dangerous" in lower_text:
            return "No"
        return "Yes"

#ADR is labelled internally within the client company

    def adr_(text):
        return "None"

    def UN(text):
        pattern = r'\bUN\s\d{4}\b'
        match = re.search(pattern, text)
        if match:
            return match.group(0)
        return None

    def supplier(text):
        lines = text.split('\n')
        supplier = None
        for i, line in enumerate(lines):
            if "Supplier" in line or "Company" in line:
                match = re.search(r'(Supplier|Company)\s*:\s*(.*)', line)
                if match and match.group(2).strip():
                    supplier = match.group(2).strip()
                else:
                    if i + 1 < len(lines) and lines[i + 1].strip() == '':
                        if i + 2 < len(lines) and lines[i + 2].strip() == ':':
                            if i + 3 < len(lines):
                                supplier = lines[i + 3].strip()
                    elif i + 1 < len(lines) and lines[i + 1].strip() == ':':
                        if i + 2 < len(lines):
                            supplier = lines[i + 2].strip()
                    elif i + 1 < len(lines):
                        supplier = lines[i + 1].strip()
                break
        return supplier
        
    def secondmethod_chemicalname(text):
        lines = [line.strip() for line in text.split('\n') if line.strip()]
        chemical_names = []
        keywords = ["CAS-No.", "EC-No.", "Index-No.", "Registration number", "Classification", "Concentration"]
        exclusion_patterns = [r'\d', r'%', r'Not Assigned', r'(\s*:\s*)', r'^\s*$', r'XXXX']
        exclude_lines = keywords + exclusion_patterns
        for i, line in enumerate(lines):
            if "Chemical name" in line:
                for j in range(i + 1, len(lines)):
                    if re.search(r'section', lines[j], re.IGNORECASE):
                        break  
                    if any(re.search(pattern, lines[j]) for pattern in exclude_lines):
                        continue
                    chemical_names.append(lines[j])
                break
        return ', '.join(chemical_names)
        
    def chemicalname(text):
        chemical_name = keyvaluepair(text, "Chemical name")
        if chemical_name == "CAS-No." or not chemical_name:
            chemical_name = secondmethod_chemicalname(text)
        return chemical_name


#1.5 This step applies the functions, adds the results to the dataframe and displays said dataframe 

    results = []

    for pdf_file in uploaded.keys():
        text = pdfcontent(pdf_file)
        product_name = keyvaluepair(text, "Product name")
        cas_no = casnumber(text)
        ec_no = ecnumber(text)
        id_uses = uses(text)
        hazard = isitahazard(text)
        adr = adr_(text)
        un_number = UN(text)
        supplier = supplier(text)
        chemical_names = chemicalname(text)

        results.append({
            'Product Name': product_name,
            'CAS Number': cas_no,
            'EC Number': ec_no,
            'Chemical Name': chemical_names,
            'Identified Uses': id_uses,
            'Supplier Name': supplier,
            'Hazard?': hazard,
            'ADR': adr,
            'UN Number': un_number
        })

    df = pd.DataFrame(results)
    display(df)

#1.6 This code creates a SQL compatible dataframe 

    from sqlalchemy import create_engine text
    lite = create_engine('sqlite://, echo = False )
    df.to_sql(name='taxonomy', con = lite)
    with engine.connect() as k:
        k.execute(text("select * from taxonomy")).fetchall()
    sqlite_taxonomy = pd.read_sql_query("select * from taxonomy", k=lite) 
    display(sqlite_taxonomy)
    
# Section 2

#2.1 First we intall the additional package of pytesseract and if have not yet intstall the packages from Section 1 

    !pip install pytesseract

    from sqlalchemy import create_engine text
    import pytesseract
    from PIL import Image
    import matplotlib.pyplot as plt

#2.2 Set up the template 
#the coordinates were manally found and entered based on a common pattern
#the hazard and un number are omitted as there was little placement pattern found wwithin the various pdfs
#the ADR is omitted because it was not included in these documents

    template = [{"key": "Product Name", "coordinates": (65, 100, 120, 110)},
    {"key": "CAS-No.", "coordinates": (70, 130, 200, 136)},
    {"key": "EC No.", "coordinates": (70, 137, 200, 147)},
    {"key": "Identified Uses", "coordinates": (20, 161, 100, 169)},
    {"key": "Supplier", "coordinates": (20, 187, 100, 195)},
    {"key": "Chemical Name", "coordinates": (10, 346, 90, 358)}]

#2.3 Make the first page an image 

    def pdftoimg(pdf_path, page_number=0, zoom=1):
        doc = fitz.open(pdf_path)
        page = doc.load_page(page_number)
        mat = fitz.Matrix(zoom, zoom)
        pix = page.get_pixmap(matrix=mat)
        img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
        return img, page, pix.width, pix.height, mat

#2.3 Upload PDF and then the template is applied and results are displayed

    def zones_img_take(pdf_path, coordinates, zoom=2, page_number=0):
        doc = fitz.open(pdf_path)
        page = doc.load_page(page_number)
        mat = fitz.Matrix(zoom, zoom)
        page_width, page_height = page.rect.width, page.rect.height
        x0, y0, x1, y1 = coordinates
        rect = fitz.Rect(x0, y0, x1, y1)
        rect = rect * mat 
        x0, y0, x1, y1 = rect.tl.x, rect.tl.y, rect.br.x, rect.br.y
        x0 = max(0, min(x0, page_width * mat.a))
        y0 = max(0, min(y0, page_height * mat.d))
        x1 = max(0, min(x1, page_width * mat.a))
        y1 = max(0, min(y1, page_height * mat.d))
        rect = fitz.Rect(x0, y0, x1, y1)
        pix = page.get_pixmap(clip=rect)
        img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
        return img, pix.width, pix.height

    uploaded = files.upload()
    pdf_path = list(uploaded.keys())[0]
    img, width, height = pdftoimg(pdf_path)
        
    for zone in template:
        key = zone['key']
        coordinates = zone['coordinates']
        extracted_img, extracted_width, extracted_height = zones_img_take(pdf_path, coordinates)
        print(f"{key}: 'Zone' size: {extracted_width} x {extracted_height} pixels")
        plt.figure(figsize=(8, 8))
        plt.imshow(extracted_img)
        plt.axis('off')
        plt.title(f'{key} Image: {extracted_width} x {extracted_height} pixels')
        plt.show()

#2.4 The information is extracted from the image into a dataframe 

    def ocr_image(image):
        return pytesseract.image_to_string(image)

    ocr_results = {}
    for zone in template:
        key = zone['key']
        coordinates = zone['coordinates']
        extracted_img, extracted_width, extracted_height = zones_img_take(pdf_path, coordinates)
        ocr_text = ocr_image(extracted_img)
        ocr_results[key] = ocr_text

    import pandas as pd
    df = pd.DataFrame([ocr_results])
    
#2.5 Dataframe can be converted into an SQL compatible format

    lite = create_engine('sqlite://, echo = False )
    df.to_sql(name='taxonomy', con = lite)
    with engine.connect() as k:
        k.execute(text("select * from taxonomy")).fetchall()
    sqlite_taxonomy = pd.read_sql_query("select * from taxonomy", k=lite) 
    display(sqlite_taxonomy)
    
# Section 3

#3.1 Install packages 

    %matplotlib widget
    
    from matplotlib.widgets import RectangleSelector
    import pandas as pd
    import matplotlib.pyplot as plt
    import matplotlib.patches as patches
    import pytesseract
    import fitz  
    import io
    from PIL import Image
    from ipywidgets import FileUpload, Button, VBox, Output
    import IPython.display as display, HTML

#3.2 Upload pdf and then convert it to an image and display (so that after the user can selecte from the image)

    upload_widget = FileUpload(accept='.pdf', multiple=False)
    def uploadfunc(change):
        for filename, file_info in change['new'].items():
            with open(filename, 'wb') as f:
                f.write(file_info['content'])
            global pdf_path
            pdf_path = filename
            display.display(display.Image(data=file_info['content'], format='png'))
            
    upload_widget.observe(uploadfunc, names='value')
    display.display(upload_widget

    upload_widget.observe(on_upload_change, names='value')
    display.display(upload_widget)

#3.3 Select using 'drag and drop' using the curser 
#The user has to make a rectangle zone around the areas they want to extract the information from 

    coordinates = []
    extractedinform = []

    def rectangleshape(ax, x0, y0, x1, y1, color='green'):
        rect = patches.Rectangle((x0, page_height - y1), x1 - x0, y1 - y0, linewidth=2, edgecolor=color, facecolor='none')
        ax.add_patch(rect)

    def usermovesmouse(eclick, erelease):
        x0, y0 = eclick.xdata, eclick.ydata
        x1, y1 = erelease.xdata, erelease.ydata
        coordinates.append((x0, y0, x1, y1))
        rectangleshape(ax, x0, y0, x1, y1)
        plt.draw()
        extract_text(x0, y0, x1, y1)

    toggle_selector = RectangleSelector(ax, usermovesmouse, interactive=True)
    plt.show()

#3.4 Click the button called 'Finish' once you are satasfied with the selected zones
#Then the extracted informtaion is displayed for verification and a dataframe is created 

    #This code instantiates a button with the label 'Finish'
    finishbutton = Button(description="Finish")
    output = Output()

    #this function ensures that after each Text (number) eg "Text 1" a new column is used 
    def createdf(extractedinform):
        info = {}
        for i, text in enumerate(extractedinform, 1):
            column = f'Text {i}'
            info[column] = [text.strip().replace('\n', ' ').replace('  ', ' ')] 
        df = pd.DataFrame(info)
        return df
 
    #This functions makes it so that when the user clicks the finish button the text is shown
    #This step is here in order to allow the user to verify the output ad see mistakes if necessary
    #It then adds the information to the dataframe, based on the pattern defined in the previos function called createdf
    
    def userclicks(b):
        toggle_selector.set_active(False)
        with output:
            for i, txt in enumerate(extractedinform, 1):
                print(f"Information number {i} below:\n{txt}\n")
            df = createdf(extractedinform)
            print("\nDataFrame:")
            display(df)
    
    finishbutton.on_click(userclicks)
    display.display(VBox([finishbutton, output]))  

#3.5 Convert df to sql compatible format

    from sqlalchemy import create_engine text
    lite = create_engine('sqlite://, echo = False )
    df.to_sql(name='taxonomy', con = lite)
    with engine.connect() as k:
        k.execute(text("select * from taxonomy")).fetchall()
    sqlite_taxonomy = pd.read_sql_query("select * from taxonomy", k=lite) 
    display(sqlite_taxonomy)



