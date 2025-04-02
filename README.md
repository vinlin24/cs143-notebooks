# Notes: Database Management Systems

Some lecture notes for the UCLA Spring 2025 offering of [COM SCI 143: Database Management Systems](https://catalog.registrar.ucla.edu/course/2024/comsci143?siteYear=current) with [Professor Remy Wang](https://remy.wang).

Class website: https://remy.wang/cs143/index.html. Note that this webpage will likely change for future offerings, just under the same URL.

I'll try to take notes live during lecture, refine them offline, and upload them to this repository. Bear with me if I lag behind a bit.

## Viewing Markdown Files on VS Code

> [!TIP]
>
> If you're viewing a source file in VS Code, you can use <kbd>Ctrl/&#8984;</kbd>+<kbd>&#8679;</kbd>+<kbd>O</kbd> to jump to a symbol in the current editor and <kbd>Ctrl/&#8984;</kbd>+<kbd>T</kbd> to jump to a symbol in the entire workspace. For Markdown, that corresponds to headers, so you can use that to preview the outline and jump around.

> [!TIP]
>
> If you're viewing these source files on VS Code, you can use <kbd>Ctrl/&#8984;</kbd>+<kbd>&#8679;</kbd>+<kbd>V</kbd> to render the Markdown in a separate tab (or <kbd>Ctrl/&#8984;</kbd>+<kbd>K</kbd> <kbd>V</kbd> to open it to the side) and read that one.

## Exporting Markdown to PDF

There are [many ways to do this](https://gist.github.com/justincbagley/ec0a6334cc86e854715e459349ab1446), but I personally use the [Markdown PDF extension](https://marketplace.visualstudio.com/items?itemName=yzane.markdown-pdf) in VS Code to export my documents. After you install the extension, simply go to the `.md` file you want to export, open the command palette (<kbd>Ctrl/&#8984;</kbd>+<kbd>&#8679;</kbd>+<kbd>P</kbd>), and search for 'Markdown PDF: Export (pdf)'.

Documents that use LaTeX expressions have a special `<script>` element appended at the end to help this extension correctly render the math expressions, something I had trouble getting to work with the other methods.

## Contributing

If you spot any errors or would like to make improvements, feel free to open an issue or pull request!
