ЛР9-10

Мигунов Т.У.

ЭВТ-70

Игровой Движок: Unity.

#### Лабараторная работа №9-10

Тема: Создание игры по типу “2048” в Unity

Цель: Разработка аналога 2048

Ход работы

1.	Настройка игрового поля 

_______![1](https://user-images.githubusercontent.com/119228138/204775784-e05d0377-f99b-436c-a62a-2ba1d8e03300.png)

_______Рис 9.1 Игровое поле

_______![3](https://user-images.githubusercontent.com/119228138/204775820-e77047da-f02e-4875-9101-a56162e63ae2.png)

 
_______Рис 9.2 Настройка игрового поля 

```2.	Скрипт Block

using System.Collections;
using System.Collections.Generic;
using TMPro;
using UnityEngine;
using static GameManager;

public class Block : MonoBehaviour
{
    public int Value;
    public Node Node;
    public Block MergingBlock;
    public bool Merging;
    public Vector2 Pos => transform.position;
    [SerializeField] private SpriteRenderer _renderer;
    [SerializeField] private TextMeshPro _text;
    public void Init(BlockType type)
    {
        Value = type.Value;
        _renderer.color = type.Color;
        _text.text = type.Value.ToString();
    }

    public void SetBlock(Node node)
    {
        if (Node != null) Node.OccupiedBlock = null;
        Node = node;
        Node.OccupiedBlock = this;
    }

    public void MergeBlock(Block blockToMergeWith)
    {
        MergingBlock = blockToMergeWith;

        Node.OccupiedBlock = null;

        blockToMergeWith.Merging = true;
    }

    public bool CanMerge(int value) => value == Value && !Merging && MergingBlock == null;
}
3.	Скрипт GameManager

public class GameManager : MonoBehaviour
{
    [SerializeField] private int _width = 4;
    [SerializeField] private int _height = 4;
    [SerializeField] private Node _nodePrefab;
    [SerializeField] private Block _blockPrefab;
    [SerializeField] private SpriteRenderer _boardPrefab;
    [SerializeField] private List<BlockType> _types;
    [SerializeField] private float _travelTime = 0.2f;
    [SerializeField] private int _winCondition = 2048;
    [SerializeField] private GameObject _winScreen, _loseScreen;
    private List<Node> _nodes;
    private List<Block> _blocks;
    private GameState _state;
    private int _round;
    private BlockType GetBlockTypeByValue(int value) => _types.First(t => t.Value == value);
    private void Start()
    {
        ChangeState(GameState.GenerateLevel);
    }
    private void ChangeState(GameState newState)
    {
        _state = newState;
        switch (newState)
        {
            case GameState.GenerateLevel:
                GenerateGrid();
                break;
            case GameState.SpawningBlock:
                SpawnBlock(_round++ == 0 ? 2 : 1);
                break;
            case GameState.WaitingInput:
                break;
            case GameState.Moving:
                break;
            case GameState.Win:
                _winScreen.SetActive(true);
                break;
            case GameState.Lose:
                _loseScreen.SetActive(true);
                break;
            default:
                break;
        }
    }
    private void Update()
    {
        if (_state != GameState.WaitingInput) return;
        if (Input.GetKeyDown(KeyCode.LeftArrow)) Shift(Vector2.left);
        if (Input.GetKeyDown(KeyCode.RightArrow)) Shift(Vector2.right);
        if (Input.GetKeyDown(KeyCode.UpArrow)) Shift(Vector2.up);
        if (Input.GetKeyDown(KeyCode.DownArrow)) Shift(Vector2.down);
    }
    private void GenerateGrid()
    {
        _round = 0;
        _nodes = new List<Node>();
        _blocks = new List<Block>();
        for (int x = 0; x < _width; x++)
        {
            for (int y = 0; y < _height; y++)
            {
                var node = Instantiate(_nodePrefab, new Vector2(x, y), Quaternion.identity);
                _nodes.Add(node);
            }
        }
        var center = new Vector2((float)_width / 2 - 0.5f, (float)_height / 2 - 0.5f);
        var board = Instantiate(_boardPrefab, center, Quaternion.identity);
        board.size = new Vector2(_width, _height);
        Camera.main.transform.position = new Vector3(center.x, center.y, -10);
        ChangeState(GameState.SpawningBlock);
    }
    private void SpawnBlock(int amount)
    {
        var freeNodes = _nodes.Where(n => n.OccupiedBlock == null).OrderBy(b => UnityEngine.Random.value);
        foreach (var node in freeNodes.Take(amount))
        {
            SpawnBlock(node, UnityEngine.Random.value > 0.8f ? 4 : 2);
        }
        if (freeNodes.Count() == 1)
        {
            ChangeState(GameState.Lose);
            return;
        }
        ChangeState(_blocks.Any(b => b.Value == _winCondition) ? GameState.Win : GameState.WaitingInput);
    }
    private void SpawnBlock(Node node, int value)
    {
        var block = Instantiate(_blockPrefab, node.Pos, Quaternion.identity);
        block.Init(GetBlockTypeByValue(value));
        block.SetBlock(node);
        _blocks.Add(block);
    }
    private void Shift(Vector2 dir)
    {
        ChangeState(GameState.Moving);
        var orderedBlock = _blocks.OrderBy(b => b.Pos.x).ThenBy(b => b.Pos.y).ToList();
        if (dir == Vector2.right || dir == Vector2.up) orderedBlock.Reverse();
        foreach (var block in orderedBlock)
        {
            var next = block.Node;
            do
            {
                block.SetBlock(next);
                var possibleNode = GetNodeAtPosition(next.Pos + dir);
                if (possibleNode != null)
                {
                    if (possibleNode.OccupiedBlock != null && possibleNode.OccupiedBlock.CanMerge(block.Value))
                    {
                        block.MergeBlock(possibleNode.OccupiedBlock);
                    }
      else if (possibleNode.OccupiedBlock == null) next = possibleNode;
                }
            } while (next != block.Node);
        }
        var sequence = DOTween.Sequence();
        foreach (var block in orderedBlock)
        {
            var movePoint = block.MergingBlock != null ? block.MergingBlock.Node.Pos : block.Node.Pos;
            sequence.Insert(0, block.transform.DOMove(movePoint, _travelTime));
        }
        sequence.OnComplete(() =>
        {
            foreach (var block in orderedBlock.Where(b => b.MergingBlock != null))
            {
                MergeBlocks(block.MergingBlock, block);
            }
            ChangeState(GameState.SpawningBlock);
        });
    }
    void MergeBlocks(Block baseBlock, Block mergingBlock)
    {
        SpawnBlock(baseBlock.Node, baseBlock.Value * 2);
        RemoveBlock(baseBlock);
        RemoveBlock(mergingBlock);
    }
    void RemoveBlock(Block block)
    {
        _blocks.Remove(block);
        Destroy(block.gameObject);
    }

    private Node GetNodeAtPosition(Vector2 pos)
    {
        return _nodes.FirstOrDefault(n => n.Pos == pos);
    }
}
[Serializable]
public struct BlockType
{
    public int Value;
    public Color Color;
}

public enum GameState
{
    GenerateLevel,
    SpawningBlock,
    WaitingInput,
    Moving,
    Win,
    Lose
}
4.	Скрипт Node
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
public class Node : MonoBehaviour
{
    public Vector2 Pos => transform.position;
    public Block OccupiedBlock;
}
```
5.	Настройка объекта GameManager
 
_______![4](https://user-images.githubusercontent.com/119228138/204775870-52020dd3-d8aa-482b-ba39-da7bc82910f3.png)
_______![5](https://user-images.githubusercontent.com/119228138/204775897-aaf28b20-e3ec-4b5f-a3a4-03f8d1350ffb.png)
 
_______Рисунки 9.3 и 9.4 Настройка объекта GameManager

6.	Настройка компонента Canvas

_______![6](https://user-images.githubusercontent.com/119228138/204776156-998481fa-4103-4e29-8998-7cbccef8c208.png)

_______Рис.9.5  Настройка компонента Canvas

7.	Настройка компонентов  WinScreen и LoseScreen  (дочерние объекты Canvas)
 
_______![7](https://user-images.githubusercontent.com/119228138/204776189-3498336c-f194-485d-a444-7322a91ca43f.png)

 
_______Рис. 9.6 Настройка WinScreen

_______![8](https://user-images.githubusercontent.com/119228138/204776271-1fc4d1de-db5f-4a27-bf15-fe7bbf0d3600.png)


_______Рис. 9.7 Настройка LoseScreen

8.	Настройка EventSystem

 _______![9](https://user-images.githubusercontent.com/119228138/204776322-d498f6aa-5f84-4bd5-b064-90629866450e.png)

 
_______Рис. 9.8 Настройка EventSystem

Вывод: В ходе проделанной работы - была разработанна игра 2048
[ЛР9,10. Разработка игры 2048.zip](https://github.com/TimurMigunov/laba9-10/files/10122300/9.10.2048.zip)
