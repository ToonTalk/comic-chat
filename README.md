# Web Comic Chat

A faithful web port of Microsoft Comic Chat (1996): type a line and it appears as a
comic panel, with the original characters, poses, balloons and panel layout.

- `index.html`: Web Comic Chat. Open it in a browser; no build step and no server.
- `comic-chat-ai.html`: the same, with AI characters to chat with.
- `making-of.html` and `comic-chat-retrospective.html`: how it was made.
- `fidelity-report.html`: how closely it follows the original.

The artwork and the layout rules come from Microsoft's open-source release,
[microsoft/ComicChat](https://github.com/microsoft/ComicChat) (characters drawn by Jim
Woodring). `engine.js` holds the layout and behavior recovered from its C++ source;
`art-original.js` is the original artwork, and `art-modern.js` a drawn alternative.

## License

MIT, see [LICENSE](LICENSE), which also carries Microsoft's notice for the artwork.
